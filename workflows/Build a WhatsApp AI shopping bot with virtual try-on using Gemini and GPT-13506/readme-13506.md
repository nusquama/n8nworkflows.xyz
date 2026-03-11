Build a WhatsApp AI shopping bot with virtual try-on using Gemini and GPT

https://n8nworkflows.xyz/workflows/build-a-whatsapp-ai-shopping-bot-with-virtual-try-on-using-gemini-and-gpt-13506


# Build a WhatsApp AI shopping bot with virtual try-on using Gemini and GPT

# 1. Workflow Overview

This workflow implements a **WhatsApp AI shopping bot for Bytez**. It receives inbound WhatsApp messages, detects whether the user is sending plain text, tapping an interactive button, or uploading an image, and then routes the request into one of three major business flows:

- **Shopping/search flow**: classify user intent with OpenAI, search products via Redis cache and MongoDB Atlas vector search, then send interactive product cards back to WhatsApp.
- **Order flow**: when the user taps **Order Now**, an AI agent orchestrates product lookup, order creation in MongoDB, logging to Google Sheets, and confirmation back to the user.
- **Virtual try-on (VTO) flow**: when the user taps **Virtual Try-On**, the workflow stores product context in Redis, asks for a selfie, validates that exactly one real person is present using Gemini, generates a try-on image with Gemini image generation, and sends the result to WhatsApp.

The workflow has a single external entry point, the **WhatsApp Trigger**, but internally it behaves like a multi-branch system with three practical entry paths:
1. **Incoming text message**
2. **Incoming interactive button tap**
3. **Incoming image message**

## 1.1 Entry Reception and Message Routing

The workflow starts from a WhatsApp trigger and immediately separates **interactive/image traffic** from **text traffic**. Interactive button taps and uploaded images are routed into the order/VTO handling branch, while text messages proceed into validation and AI classification.

## 1.2 Text Validation and Session Handling

Plain text is validated for emptiness, malformed content, spam patterns, suspicious script/XSS-like input, unsupported media types, and overlong messages. Valid text then loads or creates a Redis-backed user session and appends the user message into message history.

## 1.3 AI Classification and Conversational Routing

Validated text is sent to a LangChain AI agent backed by **GPT-5-nano** to determine whether the user wants a **product search**, **recommendation**, or just a normal conversational answer. The output is parsed into either structured JSON intent or plain text.

## 1.4 Product Search, Cache, and Vector Retrieval

If the intent is product-related, the workflow generates a Redis cache key from the interpreted search details, checks Redis first, and if no cached result exists, runs a **MongoDB Atlas vector search** using **OpenAI embeddings**. Matching products are enriched with base64-encoded product images downloaded from Google Drive and then cached.

## 1.5 Product Message Delivery

Each matched product is looped through one by one. The product image is converted to binary, uploaded to the WhatsApp media API, and then sent as an interactive WhatsApp message containing two buttons:
- **🛒 Order Now**
- **👗 Virtual Try-On**

## 1.6 Order Button Flow

If the user taps the order button, the workflow parses the button payload, extracts the product ID and user identity, and passes them to an AI order orchestration agent backed by **GPT-4o**. That agent uses MongoDB and Google Sheets tools to complete the order lifecycle and then returns a confirmation message to WhatsApp.

## 1.7 Virtual Try-On Flow

If the user taps the VTO button, the workflow stores the selected product ID in Redis for 10 minutes and prompts the user to upload a selfie. When an image arrives later, the workflow checks whether VTO context exists, downloads both the user selfie and product image, validates the selfie with Gemini, generates the try-on result with Gemini image generation, uploads the result to WhatsApp, sends it to the user, and clears Redis context.

---

# 2. Block-by-Block Analysis

## 2.1 Block: Entry Reception and Routing

### Overview
This block receives all inbound WhatsApp messages and determines the first-level execution path. It splits traffic into either the **interactive/image branch** or the **text validation branch**, then further separates button taps from user-uploaded images.

### Nodes Involved
- `WhatsApp message trigger`
- `Check if message is button or image`
- `Route interactive vs image message`

### Node Details

#### WhatsApp message trigger
- **Type and role**: `n8n-nodes-base.whatsAppTrigger`; entry point for inbound WhatsApp webhook events.
- **Configuration**:
  - Listens to `messages` updates.
- **Key data produced**:
  - `messages[0].type`
  - `contacts[0].wa_id`
  - `contacts[0].profile.name`
  - message payloads for text, image, or interactive button data.
- **Connections**:
  - Output → `Check if message is button or image`
- **Version-specific notes**:
  - Type version `1`.
  - Requires a configured WhatsApp Business integration in n8n.
- **Failure/edge cases**:
  - Webhook misconfiguration
  - Invalid WhatsApp credentials
  - Payload shape differences if Meta changes schema
  - Non-message webhook events are not handled here because only `messages` updates are selected

#### Check if message is button or image
- **Type and role**: `IF`; first-level branch router.
- **Configuration**:
  - True if `{{$json.messages[0].type}}` equals `interactive` OR `image`
  - False otherwise, which effectively means text or unsupported types go to validation
- **Connections**:
  - True → `Route interactive vs image message`
  - False → `Validate incoming message`
- **Version-specific notes**:
  - Type version `2.2`
- **Failure/edge cases**:
  - Expression failure if `messages[0]` is missing
  - Unsupported types still go to validation, where they are rejected more explicitly

#### Route interactive vs image message
- **Type and role**: `Switch`; separates interactive button taps from uploaded images.
- **Configuration**:
  - Output `Button click` if message type is `interactive`
  - Output `Image received` if message type is `image`
- **Connections**:
  - Button click → `Parse button and user data`
  - Image received → `Get VTO context from Redis`
- **Version-specific notes**:
  - Type version `3.2`
- **Failure/edge cases**:
  - Any interactive subtype outside the expected button payload may later fail in parsing
  - Image uploads without prior VTO context go to context-check fallback message

---

## 2.2 Block: Text Message Validation

### Overview
This block performs strict sanitation and validation on inbound text and interactive payloads, but in practice it is reached for non-interactive/non-image traffic. It blocks empty, meaningless, suspicious, spam-like, or unsupported messages before they reach the AI and session pipeline.

### Nodes Involved
- `Validate incoming message`
- `Pass valid messages, block invalid`
- `Send validation error to user`

### Node Details

#### Validate incoming message
- **Type and role**: `Code`; custom validation and sanitization engine.
- **Configuration choices**:
  - Detects missing payloads
  - Handles `text`, `interactive`, list and button replies, and many unsupported media types
  - Sanitizes whitespace, control chars, repeated punctuation, and bracket wrappers
  - Flags suspicious patterns such as `<script>`, `javascript:`, event handlers, iframe/object/embed, and similar HTML/script markers
  - Flags spam patterns such as excessive capitalization or repeated low-variety wording
  - Enforces maximum text length of 4096
- **Key variables created**:
  - `validation.isValid`
  - `validation.error`
  - `validation.userFriendlyError`
  - `messageText`
  - `messageType`
  - `senderId`
  - `userName`
  - `messageId`
  - `messageMetadata`
- **Connections**:
  - Output → `Pass valid messages, block invalid`
- **Version-specific notes**:
  - Type version `2`
- **Failure/edge cases**:
  - Assumes `messages[0]` structure
  - If unexpected payload shape arrives, validation gracefully marks invalid
  - Hindi/English language inference is simplistic
  - Image/audio/video/document/sticker are explicitly rejected in this path, though image messages are normally routed elsewhere first

#### Pass valid messages, block invalid
- **Type and role**: `IF`; lets only validated messages continue.
- **Configuration**:
  - True if `{{$json.validation.isValid}}` is boolean true
- **Connections**:
  - True → `Get user session from Redis`
  - False → `Send validation error to user`
- **Version-specific notes**:
  - Type version `2.2`
- **Failure/edge cases**:
  - If validation object is missing due to code changes, routing may fail

#### Send validation error to user
- **Type and role**: `WhatsApp`; sends validation failure back to user.
- **Configuration**:
  - Text body: `⚠️ Validation failed: {{ $json.validation.error }}`
  - Recipient: `{{ $('Check if message is button or image').item.json.statuses[0].recipient_id }}`
  - Disabled in current workflow
- **Connections**:
  - No downstream business logic
- **Version-specific notes**:
  - Type version `1`
- **Failure/edge cases**:
  - **Currently disabled**
  - Uses `statuses[0].recipient_id`, but the trigger path is based on message events, not status events; this may fail at runtime even if enabled
  - Better source would be `senderId` or `contacts[0].wa_id`

---

## 2.3 Block: Session Management

### Overview
This block loads the per-user conversation state from Redis, creates a new session if none exists or if the stored JSON is corrupted, appends the user message, and later appends the AI response. It limits stored history to a sliding window.

### Nodes Involved
- `Get user session from Redis`
- `Load session and append user message`
- `Append AI reply to session history`
- `Save session to Redis`

### Node Details

#### Get user session from Redis
- **Type and role**: `Redis`; retrieves the current user session string.
- **Configuration**:
  - Operation: `get`
  - Key: `{{$json.senderId}}`
  - Stores result in `sessionValue`
- **Connections**:
  - Output → `Load session and append user message`
- **Version-specific notes**:
  - Type version `1`
- **Failure/edge cases**:
  - Redis auth/connection failure
  - Missing `senderId`
  - Empty result is treated as a new session case later

#### Load session and append user message
- **Type and role**: `Code`; session loader/updater.
- **Configuration choices**:
  - Creates default session object with:
    - `userId`
    - `userName`
    - `messageHistory`
    - `createdAt`
    - `lastActivity`
    - `messageCount`
  - Parses Redis string if present
  - Falls back to new session on JSON parse failure
  - Appends current user message to `messageHistory`
  - Keeps only the last 20 messages
- **Key variables produced**:
  - `session`
  - `isNewSession`
- **Connections**:
  - Output → `AI shopping agent (BytezBot)`
- **Version-specific notes**:
  - Type version `2`
- **Failure/edge cases**:
  - Corrupted Redis session is silently replaced with a fresh one
  - Message history limit here is 20, while attached LangChain memory node uses 50-window separately; this means two different memory policies coexist

#### Append AI reply to session history
- **Type and role**: `Code`; stores assistant output in session.
- **Configuration choices**:
  - If AI output was JSON, stores a derived assistant message such as intent acknowledgement
  - If AI output was plain text, stores `responseText`
  - Keeps only last 20 messages
  - Computes `redisKey`
  - Serializes session into `updatedSessionString`
- **Key variables produced**:
  - `redisKey`
  - `updatedSessionString`
- **Connections**:
  - Output → `Save session to Redis`
- **Version-specific notes**:
  - Type version `2`
- **Failure/edge cases**:
  - If prior `session` is unexpectedly absent, node recreates a minimal fallback session
  - If sender identity cannot be resolved, creates a fallback key prefixed with `error_unknown_sender_`

#### Save session to Redis
- **Type and role**: `Redis`; persists updated session.
- **Configuration**:
  - Operation: `set`
  - Key: `{{$json.redisKey}}`
  - Value: `{{$json.updatedSessionString}}`
  - TTL enabled: `3600` seconds
- **Connections**:
  - Output → `Route JSON intent vs plain text reply`
- **Version-specific notes**:
  - Type version `1`
- **Failure/edge cases**:
  - Redis write failure
  - Missing serialized session string
  - Session expires after 1 hour of inactivity

---

## 2.4 Block: AI Intent Classification and Conversational Routing

### Overview
This block classifies the user’s validated text request into either a product-related JSON intent or a plain conversational answer. It uses a LangChain agent with GPT-5-nano plus attached memory, then parses and routes the result.

### Nodes Involved
- `GPT-5-nano (intent classifier)`
- `AI shopping agent (BytezBot)`
- `Session memory for shopping agent`
- `Detect JSON intent or plain text reply`
- `Route JSON intent vs plain text reply`
- `Route product search vs recommend`
- `Send text response to user`

### Node Details

#### GPT-5-nano (intent classifier)
- **Type and role**: `lmChatOpenAi`; language model backend for the shopping agent.
- **Configuration**:
  - Model: `gpt-5-nano`
- **Connections**:
  - AI language model → `AI shopping agent (BytezBot)`
- **Version-specific notes**:
  - Type version `1.2`
  - Requires OpenAI credentials
- **Failure/edge cases**:
  - OpenAI auth/rate limit/model availability issues
  - Model name availability depends on account and n8n/OpenAI compatibility

#### Session memory for shopping agent
- **Type and role**: `memoryBufferWindow`; memory attached to the shopping AI agent.
- **Configuration**:
  - Session key: WhatsApp user ID from trigger
  - `sessionIdType`: `customKey`
  - Context window length: `50`
- **Connections**:
  - AI memory → `AI shopping agent (BytezBot)`
- **Version-specific notes**:
  - Type version `1.3`
- **Failure/edge cases**:
  - Separate from Redis session logic; can drift from the manually stored Redis history

#### AI shopping agent (BytezBot)
- **Type and role**: `LangChain Agent`; interprets shopping intent.
- **Configuration choices**:
  - Input text: `{{ $('Validate incoming message').item.json.messages[0].text.body }}`
  - System instruction restricts role to Bytez T-shirt shopping
  - Expected outputs:
    - JSON for `product_search` or `recommend`
    - Plain text for everything else
- **Connections**:
  - Receives model from `GPT-5-nano (intent classifier)`
  - Receives memory from `Session memory for shopping agent`
  - Main output → `Detect JSON intent or plain text reply`
- **Version-specific notes**:
  - Type version `2`
- **Failure/edge cases**:
  - Uses raw trigger/body path instead of sanitized `messageText`; if validation sanitizes input, the AI node still reads original text body
  - Model may return malformed JSON despite instructions
  - Non-JSON-but-structured answers are treated as text

#### Detect JSON intent or plain text reply
- **Type and role**: `Code`; parses AI result.
- **Configuration choices**:
  - Default `responseType = text`
  - If output string starts with `{`, attempts `JSON.parse`
  - If parsed object contains `intent`, marks `responseType = json` and stores `intentData`
- **Connections**:
  - Output → `Append AI reply to session history`
- **Version-specific notes**:
  - Type version `2`
- **Failure/edge cases**:
  - Any malformed JSON silently falls back to text
  - Valid JSON without `intent` becomes a generic fallback text message

#### Route JSON intent vs plain text reply
- **Type and role**: `IF`; sends product-related JSON to search logic, everything else to direct text reply.
- **Configuration**:
  - True if `{{$json.responseType}} == 'json'`
- **Connections**:
  - True → `Route product search vs recommend`
  - False → `Send text response to user`
- **Version-specific notes**:
  - Type version `2.2`
- **Failure/edge cases**:
  - Any unexpected responseType shape will default to text path if false

#### Route product search vs recommend
- **Type and role**: `Switch`; separates product search from recommendation.
- **Configuration**:
  - Output `product_search` if `intentData.intent == 'product_search'`
  - Output `recommend` if `intentData.intent == 'recommend'`
- **Connections**:
  - Both outputs go to `Build Redis cache key from search query`
- **Version-specific notes**:
  - Type version `3.2`
- **Failure/edge cases**:
  - Recommendations are treated exactly like product search downstream; there is no distinct recommendation-specific retrieval logic
  - Any other JSON intent produces no matching route

#### Send text response to user
- **Type and role**: `WhatsApp`; sends direct conversational reply.
- **Configuration**:
  - Text body: `{{$json.responseText}}`
  - Recipient: trigger contact wa_id
- **Connections**:
  - End of plain text branch
- **Version-specific notes**:
  - Type version `1`
- **Failure/edge cases**:
  - WhatsApp API auth or delivery failure
  - If `responseText` is missing, user receives blank/invalid message

---

## 2.5 Block: Cache and Vector Product Search

### Overview
This block searches for products using a two-stage retrieval strategy: Redis cache first, then MongoDB Atlas vector search on cache miss. It also downloads product images from Google Drive, converts them to base64, combines them with product metadata, and stores the result in Redis.

### Nodes Involved
- `Build Redis cache key from search query`
- `Check Redis cache`
- `Is product cached in Redis?`
- `Parse cached product JSON array`
- `MongoDB Atlas vector search`
- `OpenAI embeddings`
- `Download product image from Drive`
- `Convert product image to base64`
- `Aggregate all product base64 images`
- `Combine vector search results with images`
- `Combine products with base64 images`
- `Store products in cache`
- `Unpack products from cache result`

### Node Details

#### Build Redis cache key from search query
- **Type and role**: `Code`; creates deterministic cache key.
- **Configuration**:
  - Reads `intentData.details`
  - Lowercases, replaces non-alphanumeric chars with `_`, truncates to 100 chars
  - Prefix: `cache:product:`
- **Key variables**:
  - `cacheKey`
- **Connections**:
  - Output → `Check Redis cache`
- **Version-specific notes**:
  - Type version `2`
- **Failure/edge cases**:
  - If `intentData.details` is missing, code can fail
  - Different semantically identical prompts may still create different keys

#### Check Redis cache
- **Type and role**: `Redis`; cache lookup.
- **Configuration**:
  - Operation: `get`
  - Key: `{{$json.cacheKey}}`
  - Result stored in `cachedProductsJson`
- **Connections**:
  - Output → `Is product cached in Redis?`
- **Failure/edge cases**:
  - Redis connectivity/auth issues
  - Large payloads may stress Redis storage depending on catalog size

#### Is product cached in Redis?
- **Type and role**: `IF`; cache hit/miss router.
- **Configuration**:
  - True if `cachedProductsJson` is not empty
- **Connections**:
  - True → `Parse cached product JSON array`
  - False → `MongoDB Atlas vector search`
- **Version-specific notes**:
  - Type version `2.2`
- **Failure/edge cases**:
  - Invalid but non-empty JSON is treated as hit and fails later during parse

#### Parse cached product JSON array
- **Type and role**: `Code`; converts cached JSON string to item list.
- **Configuration**:
  - `JSON.parse(cachedProductsJson)`
  - Returns one item per product
- **Connections**:
  - Output → `Loop through products`
- **Failure/edge cases**:
  - Runtime failure on malformed cached JSON

#### OpenAI embeddings
- **Type and role**: `embeddingsOpenAi`; embedding model provider for vector search.
- **Configuration**:
  - Default options
- **Connections**:
  - AI embedding → `MongoDB Atlas vector search`
- **Version-specific notes**:
  - Type version `1.2`
  - Requires OpenAI credentials
- **Failure/edge cases**:
  - OpenAI rate limits or auth failures

#### MongoDB Atlas vector search
- **Type and role**: `vectorStoreMongoDBAtlas`; semantic product retrieval.
- **Configuration choices**:
  - Mode: `load`
  - Top K: `3`
  - Prompt/query text: `{{ $('Build Redis cache key from search query').item.json.intentData.details }}`
  - Collection: `product`
  - Vector index name: `ShopingBot`
  - Embedding field: `text_embedding`
- **Connections**:
  - Receives embeddings from `OpenAI embeddings`
  - Main output → `Download product image from Drive`
  - Main output → `Combine vector search results with images`
- **Version-specific notes**:
  - Type version `1.3`
  - Requires MongoDB Atlas vector index named exactly `ShopingBot`
- **Failure/edge cases**:
  - Misspelled index name or absent index
  - Wrong vector field name
  - Empty search results
  - MongoDB auth/network errors

#### Download product image from Drive
- **Type and role**: `Google Drive`; downloads each matched product image.
- **Configuration**:
  - File ID: `{{$json.document.metadata.product_image_url}}`
  - Authentication: service account
  - Operation: download
- **Connections**:
  - Output → `Convert product image to base64`
- **Version-specific notes**:
  - Type version `3`
- **Failure/edge cases**:
  - Assumes `product_image_url` contains a Google Drive file ID, not a URL
  - Missing file permissions for service account
  - File missing or deleted

#### Convert product image to base64
- **Type and role**: `Extract From File`; converts downloaded binary into base64 property.
- **Configuration**:
  - Operation: `binaryToPropery`
- **Connections**:
  - Output → `Aggregate all product base64 images`
- **Failure/edge cases**:
  - If download produced no binary file, conversion fails

#### Aggregate all product base64 images
- **Type and role**: `Aggregate`; collects image data arrays.
- **Configuration**:
  - Aggregates field `data`
- **Connections**:
  - Output → `Combine vector search results with images`
- **Version-specific notes**:
  - Type version `1`
- **Failure/edge cases**:
  - Order alignment matters; assumes image aggregation order matches product result order

#### Combine vector search results with images
- **Type and role**: `Merge`; combines vector result stream and aggregated images.
- **Configuration**:
  - Mode: `combine`
  - `combineBy`: `combineAll`
- **Connections**:
  - Output → `Combine products with base64 images`
- **Version-specific notes**:
  - Type version `3.2`
- **Failure/edge cases**:
  - Potential mismatch between number/order of products and images

#### Combine products with base64 images
- **Type and role**: `Code`; constructs enriched product objects and cache payload.
- **Configuration choices**:
  - Extracts product metadata from vector results
  - Matches image arrays by index
  - Produces:
    - `products`
    - `productsJson`
    - `cacheKey`
    - `totalProducts`
    - `imagesAttached`
- **Connections**:
  - Output → `Store products in cache`
- **Failure/edge cases**:
  - If no products found, returns empty array and downstream cache/store path may stop
  - Missing image data results in `imageBase64: null`

#### Store products in cache
- **Type and role**: `Redis`; stores enriched search results.
- **Configuration**:
  - Operation: `set`
  - Key: `{{$json.cacheKey}}`
  - Value: `{{$json.productsJson}}`
  - TTL: 3600 seconds
- **Connections**:
  - Output → `Unpack products from cache result`
- **Failure/edge cases**:
  - Redis write failure
  - Large JSON values may be inefficient for cache

#### Unpack products from cache result
- **Type and role**: `Code`; re-expands stored products array into one item per product.
- **Configuration**:
  - Reads `products`
  - Returns one item per element
- **Connections**:
  - Output → `Loop through products`
- **Failure/edge cases**:
  - If `products` missing, code fails

---

## 2.6 Block: Product Message Sender

### Overview
This block iterates over matched products, builds a WhatsApp-friendly interactive card payload, uploads the product image to WhatsApp media storage, and sends each product as a button-based interactive message.

### Nodes Involved
- `Loop through products`
- `Build product card message body`
- `Convert product image base64 to binary`
- `Upload product image to WhatsApp`
- `Send product message with buttons`

### Node Details

#### Loop through products
- **Type and role**: `Split In Batches`; iterates through products sequentially.
- **Configuration**:
  - Reset: false
- **Connections**:
  - Batch output → `Build product card message body`
  - Continue loop from `Send product message with buttons`
- **Version-specific notes**:
  - Type version `3`
- **Failure/edge cases**:
  - Empty input means no outbound product messages
  - Loop relies on back-connection from send node

#### Build product card message body
- **Type and role**: `Code`; transforms product item into outbound message payload.
- **Configuration choices**:
  - Resolves recipient phone from trigger/session context
  - Chooses fields from multiple possible product schemas:
    - title/name/product_name
    - price/product_price
    - seller_name/seller
    - availability/stock_status
    - _id/product_id/id
  - Uses `discounted_price` as formatted price
  - Creates `bodyTextEscaped` via `JSON.stringify`
  - Normalizes image base64 content
- **Key variables**:
  - `to_number`
  - `bodyText`
  - `bodyTextEscaped`
  - `productId`
  - `imageBase64`
  - `imageName`
- **Connections**:
  - Output → `Convert product image base64 to binary`
- **Failure/edge cases**:
  - Uses `discounted_price`; if absent, price may be undefined even when `price` exists
  - Fallback recipient logic is complex and may mask upstream data problems

#### Convert product image base64 to binary
- **Type and role**: `Code`; prepares binary attachment for WhatsApp media API.
- **Configuration choices**:
  - If no image exists, sets `skipImageUpload: true`
  - Otherwise creates `binary.data`
- **Connections**:
  - Output → `Upload product image to WhatsApp`
- **Failure/edge cases**:
  - Downstream send node expects uploaded image id; if image is skipped, subsequent node may fail because no media id exists
  - No alternate no-image message path is implemented

#### Upload product image to WhatsApp
- **Type and role**: `HTTP Request`; uploads binary media to Meta Graph API.
- **Configuration**:
  - URL: `https://graph.facebook.com/v19.0/{{ $json.key }}/media`
  - Multipart fields:
    - `messaging_product = whatsapp`
    - `file = binary.data`
    - `type = image/jpeg`
  - Auth: predefined WhatsApp credential
  - Retry enabled
  - `onError = continueRegularOutput`
- **Connections**:
  - Output → `Send product message with buttons`
- **Failure/edge cases**:
  - **Potential issue**: URL uses `{{$json.key}}`, but no upstream node clearly sets `key` in this branch. This likely expects the WhatsApp phone-number-id or business account id and may fail unless available implicitly.
  - Media upload timeout or invalid token
  - Continues even on error, which may produce a missing `id` for next node

#### Send product message with buttons
- **Type and role**: `HTTP Request`; sends interactive product card.
- **Configuration**:
  - URL: `https://graph.facebook.com/v19.0/{{ $json.key}}/messages`
  - Sends interactive button message with:
    - Image header from uploaded media `id`
    - Body text from `bodyTextEscaped`
    - Button 1 ID: `ORDER_<productId>`
    - Button 2 ID: `VTO_<productId>`
  - Recipient `to` from `Convert product image base64 to binary`
  - Retry enabled
  - `onError = continueRegularOutput`
- **Connections**:
  - Output → back to `Loop through products`
- **Failure/edge cases**:
  - Same likely `key` issue as upload node
  - If media upload failed and no `id` exists, message send fails
  - WhatsApp interactive payload size or title constraints may apply

---

## 2.7 Block: Order Flow

### Overview
This block is triggered by an interactive button tap whose button ID starts with `ORDER_`. It extracts product and user data, then uses a GPT-4o-powered AI agent with tool access to fetch product details, create the order, log it to Google Sheets, and send the final confirmation.

### Nodes Involved
- `Parse button and user data`
- `Route VTO vs order button tap`
- `Order orchestration AI agent`
- `GPT-4o (order agent)`
- `Get product info from MongoDB`
- `Create order in MongoDB`
- `Log order to Google Sheets`
- `Session memory for order agent`
- `Send order confirmation`

### Node Details

#### Parse button and user data
- **Type and role**: `Code`; extracts action prefix and product ID from WhatsApp button payload.
- **Configuration choices**:
  - Reads `messages[0].interactive.button_reply.id`
  - Detects:
    - `VTO_...`
    - `ORDER_...`
  - Writes:
    - `prefix`
    - `productId`
    - `waId`
    - `userName`
- **Connections**:
  - Output → `Route VTO vs order button tap`
- **Failure/edge cases**:
  - Assumes `interactive.button_reply.id` exists
  - List replies are not handled in this branch
  - If unexpected button ID format, `prefix` and `productId` may remain null

#### Route VTO vs order button tap
- **Type and role**: `Switch`; routes parsed button intent.
- **Configuration**:
  - Output `VTO` if prefix = `VTO_`
  - Output `Order` if prefix = `ORDER_`
- **Connections**:
  - VTO → `Set VTO context in Redis`
  - Order → `Order orchestration AI agent`
- **Version-specific notes**:
  - Type version `3.2`
- **Failure/edge cases**:
  - Unknown prefix results in no route

#### GPT-4o (order agent)
- **Type and role**: `lmChatOpenAi`; model backend for order agent.
- **Configuration**:
  - Model: `gpt-4o`
- **Connections**:
  - AI language model → `Order orchestration AI agent`
- **Failure/edge cases**:
  - OpenAI auth/rate limit/model access issues

#### Session memory for order agent
- **Type and role**: `memoryBufferWindow`; memory for order agent.
- **Configuration**:
  - Session key: `{{$json.waId}}`
  - Context window: `50`
- **Connections**:
  - AI memory → `Order orchestration AI agent`
- **Failure/edge cases**:
  - Likely not essential for single-step order creation, but included

#### Get product info from MongoDB
- **Type and role**: `MongoDB Tool`; AI-callable lookup tool.
- **Configuration**:
  - Query uses `$fromAI('productId', ...)`
  - Collection: `product`
- **Connections**:
  - AI tool → `Order orchestration AI agent`
- **Version-specific notes**:
  - Type version `1.2`
  - Query is string-based and expects valid Object ID-like string
- **Failure/edge cases**:
  - `_id` type mismatch if MongoDB stores ObjectId rather than string
  - Not found returns incomplete downstream order composition

#### Create order in MongoDB
- **Type and role**: `MongoDB Tool`; AI-callable insert tool.
- **Configuration**:
  - Fields populated entirely through `$fromAI(...)`
  - Collection: `orders`
- **Connections**:
  - AI tool → `Order orchestration AI agent`
- **Failure/edge cases**:
  - AI must generate valid JSON string document
  - MongoDB schema mismatch or auth failure
  - Duplicate order IDs possible unless database enforces uniqueness

#### Log order to Google Sheets
- **Type and role**: `Google Sheets Tool`; AI-callable logging tool.
- **Configuration**:
  - Append operation
  - Fixed spreadsheet and `Sheet1`
  - Service account authentication
  - Columns mapped from AI-generated fields:
    - orderId, productId, productName, userName, userPhone, orderDate, status, totalAmount
- **Connections**:
  - AI tool → `Order orchestration AI agent`
- **Version-specific notes**:
  - Type version `4.7`
- **Failure/edge cases**:
  - Spreadsheet permissions for service account
  - Column name mismatch if sheet schema changes
  - Sheet ID currently hardcoded

#### Order orchestration AI agent
- **Type and role**: `LangChain Agent`; completes the full order workflow.
- **Configuration choices**:
  - Prompt includes user phone number, user name, and product ID
  - System message enforces strict sequence:
    1. Fetch product
    2. Build order document
    3. Save to MongoDB
    4. Log to Google Sheets
    5. Return confirmation
  - Tools available:
    - product lookup
    - order creation
    - Google Sheets logging
    - memory
- **Connections**:
  - Receives model from `GPT-4o (order agent)`
  - Receives tools from MongoDB/Sheets nodes
  - Receives memory from `Session memory for order agent`
  - Output → `Send order confirmation`
- **Version-specific notes**:
  - Type version `2.2`
- **Failure/edge cases**:
  - AI agent may hallucinate fields if product lookup fails
  - Tool-call sequence depends on model reliability
  - If any tool fails, only a friendly message is expected; no hard transactional rollback exists

#### Send order confirmation
- **Type and role**: `WhatsApp`; returns final order status to user.
- **Configuration**:
  - Text body from `{{ $('Order orchestration AI agent').item.json.output }}`
  - Recipient: `{{ $('Parse button and user data').item.json.waId }}`
- **Connections**:
  - End of order path
- **Failure/edge cases**:
  - If AI agent output is empty or error-shaped, user gets poor confirmation text

---

## 2.8 Block: VTO Context Setup

### Overview
This block runs when the user taps the Virtual Try-On button on a product card. It stores product context in Redis with a short TTL and prompts the user to upload a selfie.

### Nodes Involved
- `Set VTO context in Redis`
- `Prompt user to upload selfie`

### Node Details

#### Set VTO context in Redis
- **Type and role**: `Redis`; stores the selected product ID for later selfie processing.
- **Configuration**:
  - Key: `vto_context_{{ $json.waId }}`
  - Value: `{{ $json.productId }}`
  - TTL: `600` seconds
- **Connections**:
  - Output → `Prompt user to upload selfie`
- **Failure/edge cases**:
  - Redis write failure
  - Context expires after 10 minutes; later image uploads will no longer work

#### Prompt user to upload selfie
- **Type and role**: `WhatsApp`; instructs the user to send a front-facing image.
- **Configuration**:
  - Fixed message prompting clear, front-facing photo
  - Recipient from parsed button payload `waId`
- **Connections**:
  - End of VTO setup path
- **Failure/edge cases**:
  - If WhatsApp send fails, user will not know VTO context was stored

---

## 2.9 Block: VTO Context Check and Media Preparation

### Overview
When an image arrives, this block determines whether the image should be treated as a VTO selfie by checking Redis for a stored product ID. If valid context exists, it extracts image metadata and downloads both the product image and the user selfie in parallel.

### Nodes Involved
- `Get VTO context from Redis`
- `Is VTO product ID stored in Redis?`
- `Ask user to send correct photo`
- `Extract image ID, waId, and product ID`
- `Get product image from MongoDB`
- `Download product image from Drive (VTO)`
- `Resize product image to 1024px`
- `Extract product image as base64`
- `Get WhatsApp media URL`
- `Download user selfie`
- `Resize user selfie to 1024px`
- `Extract user selfie as base64`
- `Analyze user selfie with Gemini`
- `Gemini: confirm exactly one person`

### Node Details

#### Get VTO context from Redis
- **Type and role**: `Redis`; loads saved product ID for VTO.
- **Configuration**:
  - Key: `vto_context_{{ $json.contacts[0].wa_id }}`
  - Operation: get
- **Connections**:
  - Output → `Is VTO product ID stored in Redis?`
- **Failure/edge cases**:
  - Redis failure
  - Expired context leads to fallback message

#### Is VTO product ID stored in Redis?
- **Type and role**: `IF`; checks whether Redis returned a stored value.
- **Configuration**:
  - Old-style string condition on `{{$json.propertyName}} isNotEmpty`
- **Connections**:
  - True → `Extract image ID, waId, and product ID`
  - False → `Ask user to send correct photo`
- **Version-specific notes**:
  - Type version `1`
  - Uses legacy condition format unlike most newer IF nodes
- **Failure/edge cases**:
  - Relies on Redis node returning result in `propertyName`, which is the default property name here

#### Ask user to send correct photo
- **Type and role**: `WhatsApp`; fallback when user sends an image outside valid VTO context.
- **Configuration**:
  - Prompts user to send a clear, front-facing photo
  - Recipient from routed image message contact wa_id
- **Connections**:
  - End of invalid-context image path
- **Failure/edge cases**:
  - Message text implies photo-quality issue, but actual failure can simply be missing context; wording may mislead users

#### Extract image ID, waId, and product ID
- **Type and role**: `Code`; creates a compact payload for downstream media lookups.
- **Configuration**:
  - `imageId` from inbound WhatsApp image payload
  - `waId` from contact
  - `productId` from Redis VTO context
- **Connections**:
  - Output → `Get product image from MongoDB`
  - Output → `Get WhatsApp media URL`
- **Failure/edge cases**:
  - Assumes inbound message is image type with `messages[0].image.id`

#### Get product image from MongoDB
- **Type and role**: `MongoDB`; fetches selected product document.
- **Configuration**:
  - Query by `_id` equal to extracted productId
  - Collection: `product`
- **Connections**:
  - Output → `Download product image from Drive (VTO)`
- **Failure/edge cases**:
  - Same `_id` string vs ObjectId mismatch risk as order flow
  - Missing product or missing `product_image_url`

#### Download product image from Drive (VTO)
- **Type and role**: `Google Drive`; downloads garment image.
- **Configuration**:
  - File ID: `{{$json.product_image_url}}`
  - Authentication: service account
- **Connections**:
  - Output → `Resize product image to 1024px`
- **Failure/edge cases**:
  - Requires product image field to contain Drive file ID
  - Access/permission issues

#### Resize product image to 1024px
- **Type and role**: `Edit Image`; normalizes garment image dimensions.
- **Configuration**:
  - Resize to `1024 x 1024`
- **Connections**:
  - Output → `Extract product image as base64`
- **Failure/edge cases**:
  - Non-image file or corrupted binary will fail

#### Extract product image as base64
- **Type and role**: `Extract From File`; converts resized product image to property.
- **Configuration**:
  - Operation: binary to property
  - Destination key: `product_image`
- **Connections**:
  - Output → `Merge product, selfie, and validation result`
- **Failure/edge cases**:
  - Depends on successful resize

#### Get WhatsApp media URL
- **Type and role**: `HTTP Request`; resolves media download URL from image id.
- **Configuration**:
  - GET `https://graph.facebook.com/v19.0/<imageId>`
  - Auth: WhatsApp predefined credential
- **Connections**:
  - Output → `Download user selfie`
- **Failure/edge cases**:
  - WhatsApp credential/token issues
  - Media expired or inaccessible

#### Download user selfie
- **Type and role**: `HTTP Request`; downloads binary selfie.
- **Configuration**:
  - URL from previous node’s `url`
  - Response format: file
  - Output property name: `user_image`
  - Auth: WhatsApp predefined credential
- **Connections**:
  - Output → `Analyze user selfie with Gemini`
  - Output → `Resize user selfie to 1024px`
- **Failure/edge cases**:
  - Binary download failure
  - Invalid or expired media URL

#### Resize user selfie to 1024px
- **Type and role**: `Edit Image`; normalizes selfie dimensions.
- **Configuration**:
  - Resize to `1024 x 1024`
  - Data property name: `user_image`
- **Connections**:
  - Output → `Extract user selfie as base64`
- **Failure/edge cases**:
  - Corrupt or unsupported image format

#### Extract user selfie as base64
- **Type and role**: `Extract From File`; converts resized selfie into property.
- **Configuration**:
  - Destination key: `user_image`
  - Binary property name: `user_image`
- **Connections**:
  - Output → `Merge product, selfie, and validation result`
- **Failure/edge cases**:
  - Missing binary content

#### Analyze user selfie with Gemini
- **Type and role**: `Google Gemini`; image analysis for person counting.
- **Configuration**:
  - Model: `models/gemini-2.5-flash`
  - Resource: image
  - Operation: analyze
  - Binary property: `user_image`
  - Prompt forces exact output `YES` or `NO`
- **Connections**:
  - Output → `Gemini: confirm exactly one person`
- **Version-specific notes**:
  - Type version `1`
  - Requires Google Palm/Gemini API credential
- **Failure/edge cases**:
  - Model may occasionally output extra whitespace or unexpected wording despite prompt
  - Safety filtering or API quotas

#### Gemini: confirm exactly one person
- **Type and role**: `IF`; checks Gemini validation result.
- **Configuration**:
  - True if `{{$json.content.parts[0].text}} == 'YES'`
- **Connections**:
  - Both true and false are wired into `Merge product, selfie, and validation result`
- **Failure/edge cases**:
  - Because both outputs connect to the merge, validation status is not itself a branch here; later logic rechecks the same text. This is redundant and slightly confusing.
  - Case/whitespace variations could break exact match

---

## 2.10 Block: VTO Validation, Generation, and Delivery

### Overview
This block merges the garment image, user selfie, and Gemini validation result, verifies again that exactly one person was detected, builds the Gemini image-generation payload, generates the try-on result, converts it to a file, uploads it to WhatsApp, sends it to the user, and clears Redis context.

### Nodes Involved
- `Merge product, selfie, and validation result`
- `Validate merged selfie before try-on`
- `Notify user VTO is processing`
- `Ask user to resend valid photo`
- `Build Gemini image generation API payload`
- `Generate VTO with Gemini API`
- `Convert Gemini response to image file`
- `Upload VTO result to WhatsApp`
- `Send VTO result to user`
- `Delete VTO context from Redis`

### Node Details

#### Merge product, selfie, and validation result
- **Type and role**: `Merge`; combines three input streams by position.
- **Configuration**:
  - Mode: `combine`
  - Combine by: `combineByPosition`
  - Number of inputs: `3`
- **Connections**:
  - Output → `Validate merged selfie before try-on`
- **Version-specific notes**:
  - Type version `3`
- **Failure/edge cases**:
  - Positional merge assumes all three streams emit exactly corresponding items
  - If one branch emits zero or multiple items, merged record alignment may break

#### Validate merged selfie before try-on
- **Type and role**: `IF`; final go/no-go gate for generation.
- **Configuration**:
  - True if merged Gemini text is exactly `YES`
- **Connections**:
  - True → `Build Gemini image generation API payload`
  - True → `Notify user VTO is processing`
  - False → `Ask user to resend valid photo`
- **Failure/edge cases**:
  - Same exact-match fragility as earlier IF node
  - Notification and generation start in parallel; processing message may arrive after the result in some timing cases

#### Notify user VTO is processing
- **Type and role**: `WhatsApp`; progress message.
- **Configuration**:
  - Fixed text indicating generation may take a moment
  - Recipient from routed image message contact
- **Connections**:
  - No downstream dependencies
- **Failure/edge cases**:
  - Non-blocking UX node; failure does not stop generation

#### Ask user to resend valid photo
- **Type and role**: `WhatsApp`; invalid selfie feedback.
- **Configuration**:
  - Explains requirements: exactly one person, front-facing, good lighting
- **Connections**:
  - End of invalid-selfie path
- **Failure/edge cases**:
  - Does not clear VTO context, so the user can retry within TTL

#### Build Gemini image generation API payload
- **Type and role**: `Code`; constructs image generation request body.
- **Configuration choices**:
  - Reads product and user image base64 from merged item
  - Removes any `data:image/...;base64,` prefix
  - Detects mime type heuristically
  - Builds prompt:
    - apply garment from image 1 onto person in image 2
    - keep face, pose, background, and lower body unchanged
  - Sets generation config and safety settings
  - Emits `api_payload`
- **Connections**:
  - Output → `Generate VTO with Gemini API`
- **Failure/edge cases**:
  - Throws hard error if product or user image data missing
  - Assumes merged field names match expected aliases

#### Generate VTO with Gemini API
- **Type and role**: `HTTP Request`; direct Gemini image generation API call.
- **Configuration**:
  - POST to `gemini-2.5-flash-image-preview:generateContent`
  - JSON body: `{{$json.api_payload}}`
  - Header `Content-Type: application/json`
  - Auth: predefined Google Palm credential
  - Timeout: 120000 ms
  - Response format: JSON
  - Retry enabled
- **Connections**:
  - Output → `Convert Gemini response to image file`
- **Version-specific notes**:
  - Type version `4.2`
  - Uses direct HTTP instead of dedicated Gemini image-generation node
- **Failure/edge cases**:
  - API quota, safety block, timeout, or model availability errors
  - Response structure may change across Gemini preview versions

#### Convert Gemini response to image file
- **Type and role**: `Convert to File`; converts returned base64 image data to binary.
- **Configuration**:
  - Source property: `candidates[0].content.parts[0].inlineData.data`
  - Binary property name: `VTO`
- **Connections**:
  - Output → `Upload VTO result to WhatsApp`
- **Version-specific notes**:
  - Type version `1.1`
- **Failure/edge cases**:
  - Fails if Gemini response does not contain expected inlineData image payload

#### Upload VTO result to WhatsApp
- **Type and role**: `HTTP Request`; uploads generated image to WhatsApp media endpoint.
- **Configuration**:
  - URL: `https://graph.facebook.com/v19.0/{{ $json.key}}/media`
  - Multipart binary field: `VTO`
  - Auth: WhatsApp predefined credential
  - Retry enabled
- **Connections**:
  - Output → `Send VTO result to user`
- **Failure/edge cases**:
  - Same probable `key`/phone-number-id dependency issue as product media upload
  - Upload size or media format issues

#### Send VTO result to user
- **Type and role**: `HTTP Request`; sends uploaded generated image to user.
- **Configuration**:
  - URL: `https://graph.facebook.com/v19.0/{{$json.key}}/messages`
  - Image caption: `✨ Your virtual try-on result!`
  - Recipient from image-route contact wa_id
- **Connections**:
  - Output → `Delete VTO context from Redis`
- **Failure/edge cases**:
  - Depends on uploaded media `id`
  - Same likely `key` issue

#### Delete VTO context from Redis
- **Type and role**: `Redis`; clears VTO session after delivery.
- **Configuration**:
  - Delete key `vto_context_{{ $json.contacts[0].wa_id }}`
- **Connections**:
  - End of VTO success path
- **Failure/edge cases**:
  - In practice, this node may not receive original `contacts[0].wa_id` after multiple HTTP/file steps unless that field survives through the chain. This is a potential runtime bug.
  - Safer approach would be to carry `waId` explicitly through the VTO pipeline

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| WhatsApp message trigger | WhatsApp Trigger | Receives inbound WhatsApp messages |  | Check if message is button or image | ## 📥 Entry & Message Router<br>Receives all incoming WhatsApp messages. Routes button taps and image uploads to the interactive handler. Routes plain text messages to the validation pipeline.<br>## 📥 Entry & Message Router<br>Receives all incoming WhatsApp messages.<br>Routes traffic by type:<br>- Button/image → interactive handler<br>- Text → validation pipeline |
| Check if message is button or image | IF | First-level route between interactive/image and text | WhatsApp message trigger | Route interactive vs image message; Validate incoming message | ## 📥 Entry & Message Router<br>Receives all incoming WhatsApp messages. Routes button taps and image uploads to the interactive handler. Routes plain text messages to the validation pipeline.<br>## 📥 Entry & Message Router<br>Receives all incoming WhatsApp messages.<br>Routes traffic by type:<br>- Button/image → interactive handler<br>- Text → validation pipeline |
| Route interactive vs image message | Switch | Separates interactive button taps from image uploads | Check if message is button or image | Parse button and user data; Get VTO context from Redis | ## 📥 Entry & Message Router<br>Receives all incoming WhatsApp messages. Routes button taps and image uploads to the interactive handler. Routes plain text messages to the validation pipeline.<br>## 📥 Entry & Message Router<br>Receives all incoming WhatsApp messages.<br>Routes traffic by type:<br>- Button/image → interactive handler<br>- Text → validation pipeline |
| Validate incoming message | Code | Sanitizes and validates inbound text payloads | Check if message is button or image | Pass valid messages, block invalid | ## ✅ Text Message Validation<br>Validates incoming text for empty input, spam, XSS, and unsupported types. Invalid messages are rejected with a user-friendly error reply before entering the session pipeline. |
| Pass valid messages, block invalid | IF | Allows only validated text to proceed | Validate incoming message | Get user session from Redis; Send validation error to user | ## ✅ Text Message Validation<br>Validates incoming text for empty input, spam, XSS, and unsupported types. Invalid messages are rejected with a user-friendly error reply before entering the session pipeline. |
| Send validation error to user | WhatsApp | Returns validation failure to user | Pass valid messages, block invalid |  | ## ✅ Text Message Validation<br>Validates incoming text for empty input, spam, XSS, and unsupported types. Invalid messages are rejected with a user-friendly error reply before entering the session pipeline. |
| Get user session from Redis | Redis | Loads user conversation session | Pass valid messages, block invalid | Load session and append user message | ## 💾 Session Management (Redis)<br>Loads or creates a per-user Redis session (TTL: 1 hr). Appends user and assistant messages. Keeps the last 20 messages using a sliding window. |
| Load session and append user message | Code | Creates/updates Redis-backed session state | Get user session from Redis | AI shopping agent (BytezBot) | ## 💾 Session Management (Redis)<br>Loads or creates a per-user Redis session (TTL: 1 hr). Appends user and assistant messages. Keeps the last 20 messages using a sliding window. |
| GPT-5-nano (intent classifier) | OpenAI Chat Model | LLM backend for shopping agent |  | AI shopping agent (BytezBot) | ## 🧠 AI Classification & Routing<br>BytezBot classifies intent using GPT-5-nano. Returns JSON for product_search or recommend, or plain text for all other queries. Routes to the product pipeline or a direct WhatsApp reply. |
| Session memory for shopping agent | Memory Buffer Window | Agent memory for shopping assistant |  | AI shopping agent (BytezBot) | ## 🧠 AI Classification & Routing<br>BytezBot classifies intent using GPT-5-nano. Returns JSON for product_search or recommend, or plain text for all other queries. Routes to the product pipeline or a direct WhatsApp reply. |
| AI shopping agent (BytezBot) | LangChain Agent | Classifies shopping intent or generates plain reply | Load session and append user message; GPT-5-nano (intent classifier); Session memory for shopping agent | Detect JSON intent or plain text reply | ## 🧠 AI Classification & Routing<br>BytezBot classifies intent using GPT-5-nano. Returns JSON for product_search or recommend, or plain text for all other queries. Routes to the product pipeline or a direct WhatsApp reply. |
| Detect JSON intent or plain text reply | Code | Parses AI output into JSON intent or text | AI shopping agent (BytezBot) | Append AI reply to session history | ## 🧠 AI Classification & Routing<br>BytezBot classifies intent using GPT-5-nano. Returns JSON for product_search or recommend, or plain text for all other queries. Routes to the product pipeline or a direct WhatsApp reply. |
| Append AI reply to session history | Code | Adds assistant response to session history | Detect JSON intent or plain text reply | Save session to Redis | ## 💾 Session Management (Redis)<br>Loads or creates a per-user Redis session (TTL: 1 hr). Appends user and assistant messages. Keeps the last 20 messages using a sliding window.<br>## 🧠 AI Classification & Routing<br>BytezBot classifies intent using GPT-5-nano. Returns JSON for product_search or recommend, or plain text for all other queries. Routes to the product pipeline or a direct WhatsApp reply. |
| Save session to Redis | Redis | Persists updated session with TTL | Append AI reply to session history | Route JSON intent vs plain text reply | ## 💾 Session Management (Redis)<br>Loads or creates a per-user Redis session (TTL: 1 hr). Appends user and assistant messages. Keeps the last 20 messages using a sliding window. |
| Route JSON intent vs plain text reply | IF | Sends JSON intents to search and text replies to WhatsApp | Save session to Redis | Route product search vs recommend; Send text response to user | ## 🧠 AI Classification & Routing<br>BytezBot classifies intent using GPT-5-nano. Returns JSON for product_search or recommend, or plain text for all other queries. Routes to the product pipeline or a direct WhatsApp reply. |
| Route product search vs recommend | Switch | Separates product_search and recommend intents | Route JSON intent vs plain text reply | Build Redis cache key from search query | ## 🧠 AI Classification & Routing<br>BytezBot classifies intent using GPT-5-nano. Returns JSON for product_search or recommend, or plain text for all other queries. Routes to the product pipeline or a direct WhatsApp reply. |
| Send text response to user | WhatsApp | Sends direct non-product reply | Route JSON intent vs plain text reply |  | ## 🧠 AI Classification & Routing<br>BytezBot classifies intent using GPT-5-nano. Returns JSON for product_search or recommend, or plain text for all other queries. Routes to the product pipeline or a direct WhatsApp reply. |
| Build Redis cache key from search query | Code | Generates cache key from interpreted query | Route product search vs recommend | Check Redis cache | ## ⚡ Product Search (Cache + Vector)<br>Checks Redis cache first (TTL: 1 hr). On a miss, runs MongoDB Atlas vector search using OpenAI embeddings. <br>Downloads product images from Drive and stores results in cache. |
| Check Redis cache | Redis | Looks up cached search results | Build Redis cache key from search query | Is product cached in Redis? | ## ⚡ Product Search (Cache + Vector)<br>Checks Redis cache first (TTL: 1 hr). On a miss, runs MongoDB Atlas vector search using OpenAI embeddings. <br>Downloads product images from Drive and stores results in cache. |
| Is product cached in Redis? | IF | Routes cache hit vs cache miss | Check Redis cache | Parse cached product JSON array; MongoDB Atlas vector search | ## ⚡ Product Search (Cache + Vector)<br>Checks Redis cache first (TTL: 1 hr). On a miss, runs MongoDB Atlas vector search using OpenAI embeddings. <br>Downloads product images from Drive and stores results in cache. |
| Parse cached product JSON array | Code | Expands cached JSON into product items | Is product cached in Redis? | Loop through products | ## ⚡ Product Search (Cache + Vector)<br>Checks Redis cache first (TTL: 1 hr). On a miss, runs MongoDB Atlas vector search using OpenAI embeddings. <br>Downloads product images from Drive and stores results in cache. |
| OpenAI embeddings | OpenAI Embeddings | Embedding backend for vector search |  | MongoDB Atlas vector search | ## ⚡ Product Search (Cache + Vector)<br>Checks Redis cache first (TTL: 1 hr). On a miss, runs MongoDB Atlas vector search using OpenAI embeddings. <br>Downloads product images from Drive and stores results in cache. |
| MongoDB Atlas vector search | MongoDB Atlas Vector Store | Semantic product retrieval | Is product cached in Redis?; OpenAI embeddings | Download product image from Drive; Combine vector search results with images | ## ⚡ Product Search (Cache + Vector)<br>Checks Redis cache first (TTL: 1 hr). On a miss, runs MongoDB Atlas vector search using OpenAI embeddings. <br>Downloads product images from Drive and stores results in cache. |
| Download product image from Drive | Google Drive | Downloads catalog image for search result | MongoDB Atlas vector search | Convert product image to base64 | ## ⚡ Product Search (Cache + Vector)<br>Checks Redis cache first (TTL: 1 hr). On a miss, runs MongoDB Atlas vector search using OpenAI embeddings. <br>Downloads product images from Drive and stores results in cache. |
| Convert product image to base64 | Extract From File | Converts downloaded product image to base64 | Download product image from Drive | Aggregate all product base64 images | ## ⚡ Product Search (Cache + Vector)<br>Checks Redis cache first (TTL: 1 hr). On a miss, runs MongoDB Atlas vector search using OpenAI embeddings. <br>Downloads product images from Drive and stores results in cache. |
| Aggregate all product base64 images | Aggregate | Collects base64 image data from all products | Convert product image to base64 | Combine vector search results with images | ## ⚡ Product Search (Cache + Vector)<br>Checks Redis cache first (TTL: 1 hr). On a miss, runs MongoDB Atlas vector search using OpenAI embeddings. <br>Downloads product images from Drive and stores results in cache. |
| Combine vector search results with images | Merge | Merges vector results with aggregated images | MongoDB Atlas vector search; Aggregate all product base64 images | Combine products with base64 images | ## ⚡ Product Search (Cache + Vector)<br>Checks Redis cache first (TTL: 1 hr). On a miss, runs MongoDB Atlas vector search using OpenAI embeddings. <br>Downloads product images from Drive and stores results in cache. |
| Combine products with base64 images | Code | Produces enriched products array and cache payload | Combine vector search results with images | Store products in cache | ## ⚡ Product Search (Cache + Vector)<br>Checks Redis cache first (TTL: 1 hr). On a miss, runs MongoDB Atlas vector search using OpenAI embeddings. <br>Downloads product images from Drive and stores results in cache. |
| Store products in cache | Redis | Caches enriched product result set | Combine products with base64 images | Unpack products from cache result | ## ⚡ Product Search (Cache + Vector)<br>Checks Redis cache first (TTL: 1 hr). On a miss, runs MongoDB Atlas vector search using OpenAI embeddings. <br>Downloads product images from Drive and stores results in cache. |
| Unpack products from cache result | Code | Expands fresh cached array into product items | Store products in cache | Loop through products | ## ⚡ Product Search (Cache + Vector)<br>Checks Redis cache first (TTL: 1 hr). On a miss, runs MongoDB Atlas vector search using OpenAI embeddings. <br>Downloads product images from Drive and stores results in cache. |
| Loop through products | Split In Batches | Iterates through matched products | Parse cached product JSON array; Unpack products from cache result; Send product message with buttons | Build product card message body | ## 📤 Product Message Sender<br>Loops through each matched product. Uploads image to WhatsApp media API. <br>Sends an interactive message with Order Now and Virtual Try-On buttons. |
| Build product card message body | Code | Formats interactive product card content | Loop through products | Convert product image base64 to binary | ## 📤 Product Message Sender<br>Loops through each matched product. Uploads image to WhatsApp media API. <br>Sends an interactive message with Order Now and Virtual Try-On buttons. |
| Convert product image base64 to binary | Code | Prepares product image binary for WhatsApp upload | Build product card message body | Upload product image to WhatsApp | ## 📤 Product Message Sender<br>Loops through each matched product. Uploads image to WhatsApp media API. <br>Sends an interactive message with Order Now and Virtual Try-On buttons. |
| Upload product image to WhatsApp | HTTP Request | Uploads product image to WhatsApp media endpoint | Convert product image base64 to binary | Send product message with buttons | ## 📤 Product Message Sender<br>Loops through each matched product. Uploads image to WhatsApp media API. <br>Sends an interactive message with Order Now and Virtual Try-On buttons. |
| Send product message with buttons | HTTP Request | Sends interactive WhatsApp product card | Upload product image to WhatsApp | Loop through products | ## 📤 Product Message Sender<br>Loops through each matched product. Uploads image to WhatsApp media API. <br>Sends an interactive message with Order Now and Virtual Try-On buttons. |
| Parse button and user data | Code | Extracts button action, productId, and user fields | Route interactive vs image message | Route VTO vs order button tap | ## 🛒 Order Flow<br>Triggered when user taps Order Now. AI agent fetches product, creates order document, saves to MongoDB and Google Sheets, then sends a confirmation message via WhatsApp.<br>## 👗 Virtual Try-On (VTO) Flow<br>Stores product ID in Redis (TTL: 10 min) and prompts the user for a selfie. Gemini validates exactly one person, generates the try-on image, sends it via WhatsApp, then clears Redis context. |
| Route VTO vs order button tap | Switch | Splits interactive button action into VTO or order | Parse button and user data | Set VTO context in Redis; Order orchestration AI agent | ## 🛒 Order Flow<br>Triggered when user taps Order Now. AI agent fetches product, creates order document, saves to MongoDB and Google Sheets, then sends a confirmation message via WhatsApp.<br>## 👗 Virtual Try-On (VTO) Flow<br>Stores product ID in Redis (TTL: 10 min) and prompts the user for a selfie. Gemini validates exactly one person, generates the try-on image, sends it via WhatsApp, then clears Redis context. |
| Set VTO context in Redis | Redis | Stores selected productId for later selfie upload | Route VTO vs order button tap | Prompt user to upload selfie | ## 👗 Virtual Try-On (VTO) Flow<br>Stores product ID in Redis (TTL: 10 min) and prompts the user for a selfie. Gemini validates exactly one person, generates the try-on image, sends it via WhatsApp, then clears Redis context. |
| Prompt user to upload selfie | WhatsApp | Asks the user to upload a selfie | Set VTO context in Redis |  | ## 👗 Virtual Try-On (VTO) Flow<br>Stores product ID in Redis (TTL: 10 min) and prompts the user for a selfie. Gemini validates exactly one person, generates the try-on image, sends it via WhatsApp, then clears Redis context. |
| GPT-4o (order agent) | OpenAI Chat Model | LLM backend for order orchestration |  | Order orchestration AI agent | ## 🛒 Order Flow<br>Triggered when user taps Order Now. AI agent fetches product, creates order document, saves to MongoDB and Google Sheets, then sends a confirmation message via WhatsApp. |
| Session memory for order agent | Memory Buffer Window | Memory for order orchestration agent |  | Order orchestration AI agent | ## 🛒 Order Flow<br>Triggered when user taps Order Now. AI agent fetches product, creates order document, saves to MongoDB and Google Sheets, then sends a confirmation message via WhatsApp. |
| Get product info from MongoDB | MongoDB Tool | AI tool for product lookup |  | Order orchestration AI agent | ## 🛒 Order Flow<br>Triggered when user taps Order Now. AI agent fetches product, creates order document, saves to MongoDB and Google Sheets, then sends a confirmation message via WhatsApp. |
| Create order in MongoDB | MongoDB Tool | AI tool for order creation |  | Order orchestration AI agent | ## 🛒 Order Flow<br>Triggered when user taps Order Now. AI agent fetches product, creates order document, saves to MongoDB and Google Sheets, then sends a confirmation message via WhatsApp. |
| Log order to Google Sheets | Google Sheets Tool | AI tool for order logging |  | Order orchestration AI agent | ## 🛒 Order Flow<br>Triggered when user taps Order Now. AI agent fetches product, creates order document, saves to MongoDB and Google Sheets, then sends a confirmation message via WhatsApp. |
| Order orchestration AI agent | LangChain Agent | Executes tool-driven order process | Route VTO vs order button tap; GPT-4o (order agent); Get product info from MongoDB; Create order in MongoDB; Log order to Google Sheets; Session memory for order agent | Send order confirmation | ## 🛒 Order Flow<br>Triggered when user taps Order Now. AI agent fetches product, creates order document, saves to MongoDB and Google Sheets, then sends a confirmation message via WhatsApp. |
| Send order confirmation | WhatsApp | Sends final order confirmation to user | Order orchestration AI agent |  | ## 🛒 Order Flow<br>Triggered when user taps Order Now. AI agent fetches product, creates order document, saves to MongoDB and Google Sheets, then sends a confirmation message via WhatsApp. |
| Get VTO context from Redis | Redis | Retrieves pending VTO product context for image upload | Route interactive vs image message | Is VTO product ID stored in Redis? | ## 🔴 VTO Context Check<br>GETs `vto_context_{waId}` from Redis. If a product ID is stored (user previously tapped Try-On), proceed to image processing. If not found, ask user to tap the Try-On button on a product first. |
| Is VTO product ID stored in Redis? | IF | Checks whether image upload has valid pending VTO context | Get VTO context from Redis | Extract image ID, waId, and product ID; Ask user to send correct photo | ## 🔴 VTO Context Check<br>GETs `vto_context_{waId}` from Redis. If a product ID is stored (user previously tapped Try-On), proceed to image processing. If not found, ask user to tap the Try-On button on a product first. |
| Ask user to send correct photo | WhatsApp | Fallback when image is received without valid VTO context | Is VTO product ID stored in Redis? |  | ## 🔴 VTO Context Check<br>GETs `vto_context_{waId}` from Redis. If a product ID is stored (user previously tapped Try-On), proceed to image processing. If not found, ask user to tap the Try-On button on a product first. |
| Extract image ID, waId, and product ID | Code | Builds compact payload for product and selfie fetches | Is VTO product ID stored in Redis? | Get product image from MongoDB; Get WhatsApp media URL | ## 🖼️ Image Download & Preparation<br>Fetches product image from MongoDB → downloads from Google Drive → resizes to 1024×1024. In parallel, fetches the user selfie from WhatsApp media API → downloads → resizes to 1024×1024. Both images are converted to base64 for Gemini. |
| Get product image from MongoDB | MongoDB | Fetches chosen product record for VTO | Extract image ID, waId, and product ID | Download product image from Drive (VTO) | ## 🖼️ Image Download & Preparation<br>Fetches product image from MongoDB → downloads from Google Drive → resizes to 1024×1024. In parallel, fetches the user selfie from WhatsApp media API → downloads → resizes to 1024×1024. Both images are converted to base64 for Gemini. |
| Download product image from Drive (VTO) | Google Drive | Downloads selected garment image | Get product image from MongoDB | Resize product image to 1024px | ## 🖼️ Image Download & Preparation<br>Fetches product image from MongoDB → downloads from Google Drive → resizes to 1024×1024. In parallel, fetches the user selfie from WhatsApp media API → downloads → resizes to 1024×1024. Both images are converted to base64 for Gemini. |
| Resize product image to 1024px | Edit Image | Resizes product image for VTO | Download product image from Drive (VTO) | Extract product image as base64 | ## 🖼️ Image Download & Preparation<br>Fetches product image from MongoDB → downloads from Google Drive → resizes to 1024×1024. In parallel, fetches the user selfie from WhatsApp media API → downloads → resizes to 1024×1024. Both images are converted to base64 for Gemini. |
| Extract product image as base64 | Extract From File | Converts resized garment image to base64 | Resize product image to 1024px | Merge product, selfie, and validation result | ## 🖼️ Image Download & Preparation<br>Fetches product image from MongoDB → downloads from Google Drive → resizes to 1024×1024. In parallel, fetches the user selfie from WhatsApp media API → downloads → resizes to 1024×1024. Both images are converted to base64 for Gemini. |
| Get WhatsApp media URL | HTTP Request | Resolves media download URL for selfie | Extract image ID, waId, and product ID | Download user selfie | ## 🖼️ Image Download & Preparation<br>Fetches product image from MongoDB → downloads from Google Drive → resizes to 1024×1024. In parallel, fetches the user selfie from WhatsApp media API → downloads → resizes to 1024×1024. Both images are converted to base64 for Gemini. |
| Download user selfie | HTTP Request | Downloads inbound selfie binary | Get WhatsApp media URL | Analyze user selfie with Gemini; Resize user selfie to 1024px | ## 🖼️ Image Download & Preparation<br>Fetches product image from MongoDB → downloads from Google Drive → resizes to 1024×1024. In parallel, fetches the user selfie from WhatsApp media API → downloads → resizes to 1024×1024. Both images are converted to base64 for Gemini. |
| Resize user selfie to 1024px | Edit Image | Resizes selfie for generation | Download user selfie | Extract user selfie as base64 | ## 🖼️ Image Download & Preparation<br>Fetches product image from MongoDB → downloads from Google Drive → resizes to 1024×1024. In parallel, fetches the user selfie from WhatsApp media API → downloads → resizes to 1024×1024. Both images are converted to base64 for Gemini. |
| Extract user selfie as base64 | Extract From File | Converts resized selfie to base64 | Resize user selfie to 1024px | Merge product, selfie, and validation result | ## 🖼️ Image Download & Preparation<br>Fetches product image from MongoDB → downloads from Google Drive → resizes to 1024×1024. In parallel, fetches the user selfie from WhatsApp media API → downloads → resizes to 1024×1024. Both images are converted to base64 for Gemini. |
| Analyze user selfie with Gemini | Google Gemini | Validates that exactly one real person is present | Download user selfie | Gemini: confirm exactly one person | ## 🤖 Validate, Generate & Send<br><br>Gemini checks if exactly one real person is in the selfie. If valid, both images are sent to Gemini 2.5 Flash to generate the try-on result. Output is uploaded to WhatsApp and sent to the user. Redis VTO context is cleared after delivery. Invalid photos prompt a retry. |
| Gemini: confirm exactly one person | IF | Evaluates Gemini YES/NO validation result | Analyze user selfie with Gemini | Merge product, selfie, and validation result | ## 🤖 Validate, Generate & Send<br><br>Gemini checks if exactly one real person is in the selfie. If valid, both images are sent to Gemini 2.5 Flash to generate the try-on result. Output is uploaded to WhatsApp and sent to the user. Redis VTO context is cleared after delivery. Invalid photos prompt a retry. |
| Merge product, selfie, and validation result | Merge | Combines garment image, selfie image, and validation output | Extract product image as base64; Extract user selfie as base64; Gemini: confirm exactly one person | Validate merged selfie before try-on | ## 🤖 Validate, Generate & Send<br><br>Gemini checks if exactly one real person is in the selfie. If valid, both images are sent to Gemini 2.5 Flash to generate the try-on result. Output is uploaded to WhatsApp and sent to the user. Redis VTO context is cleared after delivery. Invalid photos prompt a retry. |
| Validate merged selfie before try-on | IF | Final gate before Gemini image generation | Merge product, selfie, and validation result | Build Gemini image generation API payload; Notify user VTO is processing; Ask user to resend valid photo | ## 🤖 Validate, Generate & Send<br><br>Gemini checks if exactly one real person is in the selfie. If valid, both images are sent to Gemini 2.5 Flash to generate the try-on result. Output is uploaded to WhatsApp and sent to the user. Redis VTO context is cleared after delivery. Invalid photos prompt a retry. |
| Notify user VTO is processing | WhatsApp | Sends processing/progress message | Validate merged selfie before try-on |  | ## 🤖 Validate, Generate & Send<br><br>Gemini checks if exactly one real person is in the selfie. If valid, both images are sent to Gemini 2.5 Flash to generate the try-on result. Output is uploaded to WhatsApp and sent to the user. Redis VTO context is cleared after delivery. Invalid photos prompt a retry. |
| Ask user to resend valid photo | WhatsApp | Requests new selfie when validation fails | Validate merged selfie before try-on |  | ## 🤖 Validate, Generate & Send<br><br>Gemini checks if exactly one real person is in the selfie. If valid, both images are sent to Gemini 2.5 Flash to generate the try-on result. Output is uploaded to WhatsApp and sent to the user. Redis VTO context is cleared after delivery. Invalid photos prompt a retry. |
| Build Gemini image generation API payload | Code | Builds direct Gemini image generation request | Validate merged selfie before try-on | Generate VTO with Gemini API | ## 🤖 Validate, Generate & Send<br><br>Gemini checks if exactly one real person is in the selfie. If valid, both images are sent to Gemini 2.5 Flash to generate the try-on result. Output is uploaded to WhatsApp and sent to the user. Redis VTO context is cleared after delivery. Invalid photos prompt a retry. |
| Generate VTO with Gemini API | HTTP Request | Calls Gemini image-generation API | Build Gemini image generation API payload | Convert Gemini response to image file | ## 🤖 Validate, Generate & Send<br><br>Gemini checks if exactly one real person is in the selfie. If valid, both images are sent to Gemini 2.5 Flash to generate the try-on result. Output is uploaded to WhatsApp and sent to the user. Redis VTO context is cleared after delivery. Invalid photos prompt a retry. |
| Convert Gemini response to image file | Convert To File | Converts Gemini base64 output to binary | Generate VTO with Gemini API | Upload VTO result to WhatsApp | ## 🤖 Validate, Generate & Send<br><br>Gemini checks if exactly one real person is in the selfie. If valid, both images are sent to Gemini 2.5 Flash to generate the try-on result. Output is uploaded to WhatsApp and sent to the user. Redis VTO context is cleared after delivery. Invalid photos prompt a retry. |
| Upload VTO result to WhatsApp | HTTP Request | Uploads generated try-on image to WhatsApp | Convert Gemini response to image file | Send VTO result to user | ## 🤖 Validate, Generate & Send<br><br>Gemini checks if exactly one real person is in the selfie. If valid, both images are sent to Gemini 2.5 Flash to generate the try-on result. Output is uploaded to WhatsApp and sent to the user. Redis VTO context is cleared after delivery. Invalid photos prompt a retry. |
| Send VTO result to user | HTTP Request | Sends generated try-on image to user | Upload VTO result to WhatsApp | Delete VTO context from Redis | ## 🤖 Validate, Generate & Send<br><br>Gemini checks if exactly one real person is in the selfie. If valid, both images are sent to Gemini 2.5 Flash to generate the try-on result. Output is uploaded to WhatsApp and sent to the user. Redis VTO context is cleared after delivery. Invalid photos prompt a retry. |
| Delete VTO context from Redis | Redis | Clears VTO Redis state after success | Send VTO result to user |  | ## 🤖 Validate, Generate & Send<br><br>Gemini checks if exactly one real person is in the selfie. If valid, both images are sent to Gemini 2.5 Flash to generate the try-on result. Output is uploaded to WhatsApp and sent to the user. Redis VTO context is cleared after delivery. Invalid photos prompt a retry. |
| 📌 Main overview | Sticky Note | Documentation note |  |  | ## WhatsApp AI shopping bot for Bytez<br><br>## How it works<br>1. User sends a WhatsApp message (text, button tap, or image).<br>2. GPT-5-nano classifies the intent and routes to the correct flow.<br>3. Product search checks Redis cache first, then runs MongoDB Atlas vector search using OpenAI embeddings.<br>4. Matched products are sent as interactive WhatsApp cards with Order Now and Virtual Try-On buttons.<br>5. Order tap → AI agent fetches product, creates order in MongoDB, logs to Google Sheets, sends confirmation.<br>6. Try-On tap → user sends selfie → Gemini validates one person → generates try-on image → sends result back.<br><br>## Setup steps<br>1. Add WhatsApp Business API credentials.<br>2. Add OpenAI credentials (embeddings + GPT-5-nano).<br>3. Add Google Gemini API credentials.<br>4. Add MongoDB Atlas credentials and create a vector index named `ShopingBot` on the `product` collection.<br>5. Add Redis credentials.<br>6. Add Google Drive and Sheets credentials (service account).<br>7. Import product catalog with embeddings into the MongoDB `product` collection. |
| 📌 Group: Entry & Router | Sticky Note | Documentation note |  |  | ## 📥 Entry & Message Router<br>Receives all incoming WhatsApp messages. Routes button taps and image uploads to the interactive handler. Routes plain text messages to the validation pipeline. |
| 📌 Group: Validation | Sticky Note | Documentation note |  |  | ## ✅ Text Message Validation<br>Validates incoming text for empty input, spam, XSS, and unsupported types. Invalid messages are rejected with a user-friendly error reply before entering the session pipeline. |
| 📌 Group: Session Management | Sticky Note | Documentation note |  |  | ## 💾 Session Management (Redis)<br>Loads or creates a per-user Redis session (TTL: 1 hr). Appends user and assistant messages. Keeps the last 20 messages using a sliding window. |
| 📌 Group: AI Classification | Sticky Note | Documentation note |  |  | ## 🧠 AI Classification & Routing<br>BytezBot classifies intent using GPT-5-nano. Returns JSON for product_search or recommend, or plain text for all other queries. Routes to the product pipeline or a direct WhatsApp reply. |
| 📌 Group: Cache & Vector Search | Sticky Note | Documentation note |  |  | ## ⚡ Product Search (Cache + Vector)<br>Checks Redis cache first (TTL: 1 hr). On a miss, runs MongoDB Atlas vector search using OpenAI embeddings. <br>Downloads product images from Drive and stores results in cache. |
| 📌 Group: Product Sender | Sticky Note | Documentation note |  |  | ## 📤 Product Message Sender<br>Loops through each matched product. Uploads image to WhatsApp media API. <br>Sends an interactive message with Order Now and Virtual Try-On buttons. |
| 📌 Group: Order Flow | Sticky Note | Documentation note |  |  | ## 🛒 Order Flow<br>Triggered when user taps Order Now. AI agent fetches product, creates order document, saves to MongoDB and Google Sheets, then sends a confirmation message via WhatsApp. |
| 📌 Group: VTO Flow | Sticky Note | Documentation note |  |  | ## 👗 Virtual Try-On (VTO) Flow<br>Stores product ID in Redis (TTL: 10 min) and prompts the user for a selfie. Gemini validates exactly one person, generates the try-on image, sends it via WhatsApp, then clears Redis context. |
| 📌 Group: Entry & Router1 | Sticky Note | Documentation note |  |  | ## 📥 Entry & Message Router<br>Receives all incoming WhatsApp messages.<br>Routes traffic by type:<br>- Button/image → interactive handler<br>- Text → validation pipeline |
| Group: VTO Context Check | Sticky Note | Documentation note |  |  | ## 🔴 VTO Context Check<br>GETs `vto_context_{waId}` from Redis. If a product ID is stored (user previously tapped Try-On), proceed to image processing. If not found, ask user to tap the Try-On button on a product first. |
| 0ec56bf9-33a5-44ef-bcc7-2d8154f09c85 | Sticky Note | Documentation note |  |  | ## 🖼️ Image Download & Preparation<br>Fetches product image from MongoDB → downloads from Google Drive → resizes to 1024×1024. In parallel, fetches the user selfie from WhatsApp media API → downloads → resizes to 1024×1024. Both images are converted to base64 for Gemini. |
| Group: Validate Generate Send | Sticky Note | Documentation note |  |  | ## 🤖 Validate, Generate & Send<br><br>Gemini checks if exactly one real person is in the selfie. If valid, both images are sent to Gemini 2.5 Flash to generate the try-on result. Output is uploaded to WhatsApp and sent to the user. Redis VTO context is cleared after delivery. Invalid photos prompt a retry. |

---

# 4. Reproducing the Workflow from Scratch

Below is a rebuild sequence that follows the actual logic of the workflow.

## Prerequisites

1. **Create credentials in n8n**:
   - WhatsApp Business API credential
   - OpenAI credential
   - Google Gemini / Google Palm API credential
   - Redis credential
   - MongoDB Atlas credential
   - Google Drive service account credential
   - Google Sheets service account credential

2. **Prepare external systems**:
   - MongoDB Atlas collection `product`
   - MongoDB Atlas collection `orders`
   - A vector index named **`ShopingBot`** on the `product` collection
   - Product documents should include:
     - `_id`
     - searchable metadata like title/name/details
     - embedding field `text_embedding`
     - a Google Drive file id in `product_image_url`
   - Redis reachable from n8n
   - Google Sheet with columns:
     - orderId
     - productId
     - productName
     - userName
     - userPhone
     - orderDate
     - status
     - totalAmount

3. **Know an important implementation caveat**:
   - Several HTTP WhatsApp nodes build URLs using `{{$json.key}}`. In a clean rebuild, replace that with your actual WhatsApp **phone number ID** or map it explicitly from an upstream node. Otherwise those nodes may fail.

---

## Step-by-step build

### A. Entry and routing

1. **Add a `WhatsApp Trigger` node**
   - Name: `WhatsApp message trigger`
   - Update types: `messages`
   - Connect your WhatsApp credential.

2. **Add an `If` node**
   - Name: `Check if message is button or image`
   - Condition group:
     - `{{$json.messages[0].type}} == interactive`
     - OR `{{$json.messages[0].type}} == image`
   - Connect `WhatsApp message trigger` → this node.

3. **Add a `Switch` node**
   - Name: `Route interactive vs image message`
   - Rule 1 output key: `Button click`
     - `{{$json.messages[0].type}} == interactive`
   - Rule 2 output key: `Image received`
     - `{{$json.messages[0].type}} == image`
   - Connect the **true** branch of `Check if message is button or image` to this switch.

---

### B. Validation branch for text messages

4. **Add a `Code` node**
   - Name: `Validate incoming message`
   - Paste the validation code from the workflow logic:
     - validates text emptiness
     - sanitizes text
     - detects suspicious content and spam
     - fills `validation`, `messageText`, `senderId`, `userName`, etc.
   - Connect the **false** branch of `Check if message is button or image` to this node.

5. **Add an `If` node**
   - Name: `Pass valid messages, block invalid`
   - Condition:
     - `{{$json.validation.isValid}}` is true
   - Connect `Validate incoming message` → this node.

6. **Add a `WhatsApp` send node**
   - Name: `Send validation error to user`
   - Operation: `send`
   - Text body: use a user-friendly validation message
   - Recipient: prefer `{{$json.senderId}}`
   - Phone number ID: your WhatsApp phone number ID
   - In the source workflow this node is disabled; keep it disabled if you want exact parity, or enable it with corrected recipient logic.

---

### C. Session management

7. **Add a `Redis` node**
   - Name: `Get user session from Redis`
   - Operation: `get`
   - Key: `{{$json.senderId}}`
   - Output property: `sessionValue`
   - Connect the **true** branch of `Pass valid messages, block invalid` to this node.

8. **Add a `Code` node**
   - Name: `Load session and append user message`
   - Logic:
     - parse `sessionValue` if present
     - create new session if missing/corrupt
     - append user message to `session.messageHistory`
     - keep last 20 messages
   - Connect from `Get user session from Redis`.

---

### D. Shopping AI classification

9. **Add an `OpenAI Chat Model` node**
   - Name: `GPT-5-nano (intent classifier)`
   - Model: `gpt-5-nano`
   - Connect OpenAI credentials.

10. **Add a `Memory Buffer Window` node**
   - Name: `Session memory for shopping agent`
   - Session key: `{{$('WhatsApp message trigger').item.json.contacts[0].wa_id}}`
   - Session ID type: `customKey`
   - Context window length: `50`

11. **Add a `LangChain Agent` node**
   - Name: `AI shopping agent (BytezBot)`
   - Prompt type: define
   - User text:
     - either replicate exact original: `{{$('Validate incoming message').item.json.messages[0].text.body}}`
     - or better: `{{$json.messageText}}`
   - System message should instruct:
     - product-related outputs must be JSON:
       - `{"intent":"product_search","details":"...","confidence":0.95,"language":"english"}`
     - all other replies should be plain text
     - allowed product intents: `product_search`, `recommend`
   - Connect:
     - `Load session and append user message` → agent main input
     - `GPT-5-nano (intent classifier)` → agent language model
     - `Session memory for shopping agent` → agent memory

12. **Add a `Code` node**
   - Name: `Detect JSON intent or plain text reply`
   - Logic:
     - read `output`
     - if it looks like JSON and contains `intent`, set `responseType = 'json'`
     - else set `responseType = 'text'` and `responseText`
   - Connect from `AI shopping agent (BytezBot)`.

13. **Add a `Code` node**
   - Name: `Append AI reply to session history`
   - Logic:
     - append assistant summary or response text into `session.messageHistory`
     - keep last 20
     - build `redisKey`
     - serialize to `updatedSessionString`
   - Connect from `Detect JSON intent or plain text reply`.

14. **Add a `Redis` node**
   - Name: `Save session to Redis`
   - Operation: `set`
   - Key: `{{$json.redisKey}}`
   - Value: `{{$json.updatedSessionString}}`
   - TTL enabled: 3600
   - Connect from `Append AI reply to session history`.

15. **Add an `If` node**
   - Name: `Route JSON intent vs plain text reply`
   - Condition:
     - `{{$json.responseType}} == json`
   - Connect from `Save session to Redis`.

16. **Add a `Switch` node**
   - Name: `Route product search vs recommend`
   - Output 1: `product_search`
     - `{{$json.intentData.intent}} == product_search`
   - Output 2: `recommend`
     - `{{$json.intentData.intent}} == recommend`
   - Connect from the **true** branch of `Route JSON intent vs plain text reply`.

17. **Add a `WhatsApp` send node**
   - Name: `Send text response to user`
   - Operation: `send`
   - Text body: `{{$json.responseText}}`
   - Recipient: `{{$('WhatsApp message trigger').item.json.contacts[0].wa_id}}`
   - Phone number ID: your WhatsApp phone number ID
   - Connect from the **false** branch of `Route JSON intent vs plain text reply`.

---

### E. Product search with Redis cache and MongoDB vector search

18. **Add a `Code` node**
   - Name: `Build Redis cache key from search query`
   - Logic:
     - read `intentData.details`
     - normalize lower-case alphanumeric slug
     - prefix with `cache:product:`
     - store as `cacheKey`
   - Connect both outputs of `Route product search vs recommend` to this node.

19. **Add a `Redis` node**
   - Name: `Check Redis cache`
   - Operation: `get`
   - Key: `{{$json.cacheKey}}`
   - Property name: `cachedProductsJson`
   - Connect from `Build Redis cache key from search query`.

20. **Add an `If` node**
   - Name: `Is product cached in Redis?`
   - Condition:
     - `{{$json.cachedProductsJson}}` is not empty
   - Connect from `Check Redis cache`.

21. **Add a `Code` node**
   - Name: `Parse cached product JSON array`
   - Logic:
     - parse `cachedProductsJson`
     - return one item per product
   - Connect from the **true** branch of `Is product cached in Redis?`.

22. **Add an `OpenAI Embeddings` node**
   - Name: `OpenAI embeddings`
   - Connect OpenAI credentials.

23. **Add a `MongoDB Atlas Vector Store` node**
   - Name: `MongoDB Atlas vector search`
   - Mode: `load`
   - Collection: `product`
   - Vector index name: `ShopingBot`
   - Embedding field: `text_embedding`
   - Top K: `3`
   - Prompt/query text: `{{$('Build Redis cache key from search query').item.json.intentData.details}}`
   - Connect:
     - **false** branch of `Is product cached in Redis?` → main input
     - `OpenAI embeddings` → AI embedding input

24. **Add a `Google Drive` node**
   - Name: `Download product image from Drive`
   - Operation: `download`
   - Authentication: `serviceAccount`
   - File ID: `{{$json.document.metadata.product_image_url}}`
   - Connect from `MongoDB Atlas vector search`.

25. **Add an `Extract From File` node**
   - Name: `Convert product image to base64`
   - Operation: `binaryToPropery`
   - Connect from `Download product image from Drive`.

26. **Add an `Aggregate` node**
   - Name: `Aggregate all product base64 images`
   - Aggregate field: `data`
   - Connect from `Convert product image to base64`.

27. **Add a `Merge` node**
   - Name: `Combine vector search results with images`
   - Mode: `combine`
   - Combine by: `combineAll`
   - Input 1: from `MongoDB Atlas vector search`
   - Input 2: from `Aggregate all product base64 images`

28. **Add a `Code` node**
   - Name: `Combine products with base64 images`
   - Logic:
     - read vector results from `document.metadata`
     - attach base64 images by index
     - emit:
       - `products`
       - `productsJson`
       - `cacheKey`
   - Connect from `Combine vector search results with images`.

29. **Add a `Redis` node**
   - Name: `Store products in cache`
   - Operation: `set`
   - Key: `{{$json.cacheKey}}`
   - Value: `{{$json.productsJson}}`
   - TTL: `3600`
   - Connect from `Combine products with base64 images`.

30. **Add a `Code` node**
   - Name: `Unpack products from cache result`
   - Logic:
     - loop over `products`
     - emit one item per product
   - Connect from `Store products in cache`.

---

### F. Send product cards

31. **Add a `Split In Batches` node**
   - Name: `Loop through products`
   - Default batch behavior is sufficient
   - Connect both:
     - `Parse cached product JSON array`
     - `Unpack products from cache result`
     to this node.

32. **Add a `Code` node**
   - Name: `Build product card message body`
   - Logic:
     - create body with product name, seller, availability, price
     - prepare `bodyTextEscaped`
     - set `productId`
     - normalize image base64
     - preserve destination phone number
   - Connect from `Loop through products`.

33. **Add a `Code` node**
   - Name: `Convert product image base64 to binary`
   - Logic:
     - if no base64, mark `skipImageUpload`
     - else output `binary.data`
   - Connect from `Build product card message body`.

34. **Add an `HTTP Request` node**
   - Name: `Upload product image to WhatsApp`
   - Method: `POST`
   - URL:
     - use your WhatsApp phone number ID, e.g. `https://graph.facebook.com/v19.0/<PHONE_NUMBER_ID>/media`
     - do not rely on `{{$json.key}}` unless you intentionally inject it
   - Authentication: WhatsApp predefined credential
   - Content type: `multipart-form-data`
   - Form fields:
     - `messaging_product = whatsapp`
     - `file = binary.data`
     - `type = image/jpeg`
   - Retry on fail: enabled
   - On error: continue regular output
   - Connect from `Convert product image base64 to binary`.

35. **Add an `HTTP Request` node**
   - Name: `Send product message with buttons`
   - Method: `POST`
   - URL:
     - `https://graph.facebook.com/v19.0/<PHONE_NUMBER_ID>/messages`
   - Authentication: WhatsApp predefined credential
   - JSON body:
     - `messaging_product = whatsapp`
     - `to = {{$('Convert product image base64 to binary').first().json.to_number}}`
     - `type = interactive`
     - interactive header image uses uploaded media `id`
     - body text uses `bodyTextEscaped`
     - buttons:
       - `ORDER_<productId>`
       - `VTO_<productId>`
   - Retry on fail: enabled
   - On error: continue regular output
   - Connect from `Upload product image to WhatsApp`.

36. **Close the product loop**
   - Connect `Send product message with buttons` back to `Loop through products` continuation input.

---

### G. Parse interactive buttons

37. **Add a `Code` node**
   - Name: `Parse button and user data`
   - Logic:
     - read `messages[0].interactive.button_reply.id`
     - if prefix is `VTO_`, extract product ID
     - if prefix is `ORDER_`, extract product ID
     - also set `waId` and `userName`
   - Connect `Route interactive vs image message` output `Button click` to this node.

38. **Add a `Switch` node**
   - Name: `Route VTO vs order button tap`
   - Output `VTO` if `prefix == VTO_`
   - Output `Order` if `prefix == ORDER_`
   - Connect from `Parse button and user data`.

---

### H. Order flow

39. **Add a `Memory Buffer Window` node**
   - Name: `Session memory for order agent`
   - Session key: `{{$json.waId}}`
   - Session ID type: `customKey`
   - Context window: `50`

40. **Add an `OpenAI Chat Model` node**
   - Name: `GPT-4o (order agent)`
   - Model: `gpt-4o`

41. **Add a `MongoDB Tool` node**
   - Name: `Get product info from MongoDB`
   - Collection: `product`
   - Query:
     - lookup by `_id` from `$fromAI('productId', ...)`
   - If your `_id` is true `ObjectId`, adjust query accordingly.

42. **Add a `MongoDB Tool` node**
   - Name: `Create order in MongoDB`
   - Collection: `orders`
   - Operation: `insert`
   - Fields should be supplied by `$fromAI(...)`

43. **Add a `Google Sheets Tool` node**
   - Name: `Log order to Google Sheets`
   - Operation: `append`
   - Spreadsheet: your order log sheet
   - Sheet: your target tab
   - Authentication: `serviceAccount`
   - Map columns:
     - orderId
     - productId
     - productName
     - userName
     - userPhone
     - orderDate
     - status
     - totalAmount

44. **Add a `LangChain Agent` node**
   - Name: `Order orchestration AI agent`
   - Prompt should tell the agent:
     - fetch product info
     - create order document
     - insert into MongoDB
     - log to Google Sheets
     - reply with final confirmation
     - do not ask the user any follow-up questions
   - Main input text:
     - include `waId`, `userName`, and `productId`
   - Connect:
     - `Route VTO vs order button tap` output `Order` → main input
     - `GPT-4o (order agent)` → language model
     - `Get product info from MongoDB` → tool
     - `Create order in MongoDB` → tool
     - `Log order to Google Sheets` → tool
     - `Session memory for order agent` → memory

45. **Add a `WhatsApp` send node**
   - Name: `Send order confirmation`
   - Operation: `send`
   - Text body: `{{$('Order orchestration AI agent').item.json.output}}`
   - Recipient: `{{$('Parse button and user data').item.json.waId}}`
   - Phone number ID: your WhatsApp phone number ID
   - Connect from `Order orchestration AI agent`.

---

### I. VTO setup after button tap

46. **Add a `Redis` node**
   - Name: `Set VTO context in Redis`
   - Operation: `set`
   - Key: `vto_context_{{$json.waId}}`
   - Value: `{{$json.productId}}`
   - TTL: `600`
   - Connect from `Route VTO vs order button tap` output `VTO`.

47. **Add a `WhatsApp` send node**
   - Name: `Prompt user to upload selfie`
   - Operation: `send`
   - Recipient: `{{$('Parse button and user data').item.json.waId}}`
   - Text: prompt for clear front-facing selfie
   - Phone number ID: your WhatsApp phone number ID
   - Connect from `Set VTO context in Redis`.

---

### J. VTO context retrieval when image arrives

48. **Add a `Redis` node**
   - Name: `Get VTO context from Redis`
   - Operation: `get`
   - Key: `vto_context_{{$json.contacts[0].wa_id}}`
   - Connect `Route interactive vs image message` output `Image received` to this node.

49. **Add an `If` node**
   - Name: `Is VTO product ID stored in Redis?`
   - Check that the returned Redis property is not empty.
   - Connect from `Get VTO context from Redis`.

50. **Add a `WhatsApp` send node**
   - Name: `Ask user to send correct photo`
   - Operation: `send`
   - Recipient: `{{$('Route interactive vs image message').item.json.contacts[0].wa_id}}`
   - Text: instruct user to use the try-on feature correctly
   - Connect from the **false** branch of `Is VTO product ID stored in Redis?`.

51. **Add a `Code` node**
   - Name: `Extract image ID, waId, and product ID`
   - Logic:
     - imageId from inbound image
     - waId from contact
     - productId from Redis context value
   - Connect from the **true** branch of `Is VTO product ID stored in Redis?`.

---

### K. Download product image and user selfie

52. **Add a `MongoDB` node**
   - Name: `Get product image from MongoDB`
   - Collection: `product`
   - Query by `_id` = extracted `productId`
   - Connect from `Extract image ID, waId, and product ID`.

53. **Add a `Google Drive` node**
   - Name: `Download product image from Drive (VTO)`
   - Operation: `download`
   - Authentication: service account
   - File ID: `{{$json.product_image_url}}`
   - Connect from `Get product image from MongoDB`.

54. **Add an `Edit Image` node**
   - Name: `Resize product image to 1024px`
   - Operation: resize
   - Width: `1024`
   - Height: `1024`
   - Connect from `Download product image from Drive (VTO)`.

55. **Add an `Extract From File` node**
   - Name: `Extract product image as base64`
   - Operation: `binaryToPropery`
   - Destination key: `product_image`
   - Connect from `Resize product image to 1024px`.

56. **Add an `HTTP Request` node**
   - Name: `Get WhatsApp media URL`
   - Method: `GET`
   - URL: `https://graph.facebook.com/v19.0/{{$('Extract image ID, waId, and product ID').item.json.imageId}}`
   - Authentication: WhatsApp predefined credential
   - Connect from `Extract image ID, waId, and product ID`.

57. **Add an `HTTP Request` node**
   - Name: `Download user selfie`
   - Method: `GET`
   - URL: `{{$json.url}}`
   - Authentication: WhatsApp predefined credential
   - Response format: `file`
   - Output property: `user_image`
   - Connect from `Get WhatsApp media URL`.

58. **Add a `Google Gemini` node**
   - Name: `Analyze user selfie with Gemini`
   - Resource: `image`
   - Operation: `analyze`
   - Input type: `binary`
   - Binary property: `user_image`
   - Model: `models/gemini-2.5-flash`
   - Prompt must force exact response:
     - `YES` if exactly one real human
     - `NO` otherwise
   - Connect from `Download user selfie`.

59. **Add an `If` node**
   - Name: `Gemini: confirm exactly one person`
   - Condition:
     - `{{$json.content.parts[0].text}} == YES`
   - Connect from `Analyze user selfie with Gemini`.

60. **Add an `Edit Image` node**
   - Name: `Resize user selfie to 1024px`
   - Operation: resize
   - Width: `1024`
   - Height: `1024`
   - Data property name: `user_image`
   - Connect from `Download user selfie`.

61. **Add an `Extract From File` node**
   - Name: `Extract user selfie as base64`
   - Operation: `binaryToPropery`
   - Binary property name: `user_image`
   - Destination key: `user_image`
   - Connect from `Resize user selfie to 1024px`.

---

### L. Merge, validate, generate, and send VTO result

62. **Add a `Merge` node**
   - Name: `Merge product, selfie, and validation result`
   - Mode: `combine`
   - Combine by: `combineByPosition`
   - Number of inputs: `3`
   - Connect:
     - `Extract product image as base64`
     - `Extract user selfie as base64`
     - `Gemini: confirm exactly one person`
   - For exact parity, both true and false outputs from the `Gemini` IF feed this merge.

63. **Add an `If` node**
   - Name: `Validate merged selfie before try-on`
   - Condition:
     - `{{$json.content.parts[0].text}} == YES`
   - Connect from `Merge product, selfie, and validation result`.

64. **Add a `WhatsApp` send node**
   - Name: `Notify user VTO is processing`
   - Operation: `send`
   - Recipient: `{{$('Route interactive vs image message').item.json.contacts[0].wa_id}}`
   - Text: processing message
   - Connect from the **true** branch of `Validate merged selfie before try-on`.

65. **Add a `WhatsApp` send node**
   - Name: `Ask user to resend valid photo`
   - Operation: `send`
   - Recipient: `{{$('Route interactive vs image message').item.json.contacts[0].wa_id}}`
   - Text: explain exactly-one-person rule
   - Connect from the **false** branch.

66. **Add a `Code` node**
   - Name: `Build Gemini image generation API payload`
   - Logic:
     - clean base64 strings
     - detect mime types
     - build prompt instructing garment transfer from image 1 to image 2
     - create `api_payload` object with generationConfig and safetySettings
   - Connect from the **true** branch of `Validate merged selfie before try-on`.

67. **Add an `HTTP Request` node**
   - Name: `Generate VTO with Gemini API`
   - Method: `POST`
   - URL:
     - `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-image-preview:generateContent`
   - Auth: Google Palm/Gemini predefined credential
   - Header: `Content-Type: application/json`
   - Send JSON body: `{{$json.api_payload}}`
   - Timeout: `120000`
   - Retry on fail: enabled
   - Response format: JSON
   - Connect from `Build Gemini image generation API payload`.

68. **Add a `Convert To File` node**
   - Name: `Convert Gemini response to image file`
   - Operation: `toBinary`
   - Source property: `candidates[0].content.parts[0].inlineData.data`
   - Binary property name: `VTO`
   - Connect from `Generate VTO with Gemini API`.

69. **Add an `HTTP Request` node**
   - Name: `Upload VTO result to WhatsApp`
   - Method: `POST`
   - URL:
     - `https://graph.facebook.com/v19.0/<PHONE_NUMBER_ID>/media`
   - Authentication: WhatsApp predefined credential
   - Multipart form fields:
     - `messaging_product = whatsapp`
     - `file = binary.VTO`
     - `type = image/jpeg`
   - Retry on fail: enabled
   - Connect from `Convert Gemini response to image file`.

70. **Add an `HTTP Request` node**
   - Name: `Send VTO result to user`
   - Method: `POST`
   - URL:
     - `https://graph.facebook.com/v19.0/<PHONE_NUMBER_ID>/messages`
   - Authentication: WhatsApp predefined credential
   - JSON body:
     - type `image`
     - image id from uploaded media
     - caption `✨ Your virtual try-on result!`
     - recipient = routed image sender wa_id
   - Connect from `Upload VTO result to WhatsApp`.

71. **Add a `Redis` node**
   - Name: `Delete VTO context from Redis`
   - Operation: `delete`
   - Key:
     - ideally `vto_context_{{$('Extract image ID, waId, and product ID').item.json.waId}}`
   - Connect from `Send VTO result to user`.
   - This is safer than the original expression, which may lose `contacts[0].wa_id` by that stage.

---

## Suggested corrections if rebuilding for production

To make the workflow more robust than the source version:

1. **Use explicit WhatsApp phone number ID**
   - Replace all `{{$json.key}}` in Graph API URLs with a fixed expression or stored variable.

2. **Use sanitized text for AI**
   - In `AI shopping agent (BytezBot)`, prefer `{{$json.messageText}}` over the raw original text body.

3. **Fix validation error recipient**
   - Use `{{$json.senderId}}`, not `statuses[0].recipient_id`.

4. **Carry `waId` through the full VTO pipeline**
   - Avoid relying on original trigger payload after multiple HTTP/file nodes.

5. **Handle no-image product cards**
   - Add a fallback text-only interactive or non-interactive message if image upload fails.

6. **Handle Mongo `_id` properly**
   - If your collections use true `ObjectId`, adapt Mongo queries accordingly.

7. **Separate recommend logic if desired**
   - Right now `recommend` and `product_search` both reuse the same retrieval path.

---

## No sub-workflows

This workflow does **not** invoke any separate n8n sub-workflows. All logic is contained within a single workflow.

If you want, I can also provide:
- a **node dependency map**
- a **mermaid flow diagram**
- or a **production hardening review** of this workflow.
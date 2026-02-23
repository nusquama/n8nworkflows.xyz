Send WhatsApp new product campaigns from WooCommerce with OpenAI and Sheets

https://n8nworkflows.xyz/workflows/send-whatsapp-new-product-campaigns-from-woocommerce-with-openai-and-sheets-13416


# Send WhatsApp new product campaigns from WooCommerce with OpenAI and Sheets

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow listens for **new product events in WooCommerce**, extracts/filters the product’s image links, uses **OpenAI** to shorten the product’s HTML description into a campaign-friendly text, then pulls **all WooCommerce customers**, verifies each customer’s WhatsApp number via **Rapiwa**, sends a WhatsApp campaign message to verified numbers, and finally logs outcomes to **Google Sheets** (sent vs. not sent/unverified). A **Wait** node throttles/controls pacing between iterations.

**Target use cases:**
- Automated “new product launch” WhatsApp broadcasts to existing WooCommerce customers
- Lightweight campaign logging/auditing in Google Sheets
- Basic number validation before sending WhatsApp messages

### Logical Blocks
1.1 **Trigger & Product Data Preparation** (WooCommerce trigger → image extraction → image validation → per-image loop)  
1.2 **AI Copy Generation** (OpenAI short description creation)  
1.3 **Customer Retrieval & Cleanup** (WooCommerce customers → normalization/cleanup → batching)  
1.4 **WhatsApp Verification, Sending & Logging** (verify → route by verification → send or log unverified → wait)

---

## 2. Block-by-Block Analysis

### 2.1 Trigger & Product Data Preparation

**Overview:**  
Receives a WooCommerce event (intended for “new product”), extracts image links, and filters them to keep only valid image files. If no valid images are found, it sends a WhatsApp notification about the missing images.

**Nodes Involved:**
- WooCommerce Trigger
- Code (image link detect)
- IF – Keeps only valid image files
- Loop Over Image Link
- Rapiwa (WhatsApp Notify No images found for the product)

#### Node: WooCommerce Trigger
- **Type / role:** `n8n-nodes-base.wooCommerceTrigger` — Webhook-based trigger from WooCommerce.
- **Configuration (interpreted):** Uses a WooCommerce webhook subscription (event type not visible in JSON) with `webhookId` stored in n8n.
- **Input/Output:** Entry point → outputs WooCommerce payload to `Code (image link detect)`.
- **Version notes:** Type version `1`.
- **Potential failures / edge cases:**
  - WooCommerce webhook not registered or disabled
  - Signature/secret mismatch (if configured)
  - Payload missing expected product fields (images, description, etc.)

#### Node: Code (image link detect)
- **Type / role:** `n8n-nodes-base.code` — Custom parsing to detect/extract image links from the incoming product payload.
- **Configuration choices:** Not provided in JSON (empty parameters). Intended to output an array/list of image URLs (or items with image URL fields).
- **Key variables/expressions:** Not visible. Typically would reference `$json` product fields like `images`, `images[].src`, etc.
- **Input/Output:** From trigger → to `IF – Keeps only valid image files`.
- **Version notes:** Code node type version `2` (newer runtime semantics vs v1).
- **Failure modes:**
  - JavaScript runtime errors (undefined fields)
  - Producing no items / malformed structure for downstream IF and batching

#### Node: IF – Keeps only valid image files
- **Type / role:** `n8n-nodes-base.if` — Branching to keep only valid image file links.
- **Configuration choices:** Rules not included in JSON. Intended to check extension/mimetype (e.g., `.jpg`, `.png`, `.webp`) and/or URL validity.
- **Input/Output connections:**
  - **True branch (index 0):** → `Loop Over Image Link` (valid images exist)
  - **False branch (index 1):** → `Rapiwa (WhatsApp Notify No images found for the product)` (no valid images)
- **Version notes:** `2.2`.
- **Edge cases:**
  - If code node outputs a single item with an array field, IF may not “see” each image unless data is itemized first
  - File extensions with query params (`.jpg?x=...`) might fail naive checks

#### Node: Loop Over Image Link
- **Type / role:** `n8n-nodes-base.splitInBatches` — Iterates over image-link items in batches (typically batch size = 1).
- **Configuration choices:** Not provided in JSON (so defaults apply unless set in UI).
- **Connections:**
  - Receives valid image items from the IF node
  - Its **“continue” output (index 1)** goes to `Create the product description (HTML) into a short`
  - The first output is unused here (common when the node is used in “manual loop” patterns, but this wiring is unusual and may indicate a misconnection or a specific batching mode expectation).
- **Version notes:** `3`.
- **Edge cases:**
  - If batch size is too large, you may send too many images / exceed WhatsApp or template constraints later
  - If miswired, the loop may not behave as expected (see “Reproducing” section recommendations)

#### Node: Rapiwa (WhatsApp Notify No images found for the product)
- **Type / role:** `n8n-nodes-rapiwa.rapiwa` — Sends a WhatsApp alert when no images are found/kept.
- **Configuration choices:** Operation not visible; likely “send message” to an admin/owner number.
- **Input/Output:** Receives from IF false branch; no downstream nodes.
- **Failure modes:**
  - Rapiwa credential/auth errors
  - Destination number missing/invalid
  - Provider rate limits or downtime

---

### 2.2 AI Copy Generation

**Overview:**  
Uses OpenAI (via the n8n LangChain OpenAI node) to convert the product HTML description into a shorter campaign-ready text, then passes control to customer retrieval.

**Nodes Involved:**
- Create the product description (HTML) into a short

#### Node: Create the product description (HTML) into a short
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` — LLM call to generate short marketing text from HTML product description.
- **Configuration choices:** Not visible in JSON. Common configuration would include:
  - Model (e.g., GPT-4o-mini or similar)
  - Prompt instructions (“Summarize HTML description into short WhatsApp-friendly copy”)
  - Possibly variables referencing `$json.description` or similar fields from WooCommerce trigger.
- **Input/Output connections:** From `Loop Over Image Link` → to `Get All Customer records from the WooCommerce store`.
- **Version notes:** Type version `1.8` (LangChain-based OpenAI node).
- **Edge cases / failures:**
  - Missing product description field
  - Token limits if HTML is large
  - Model/credential misconfiguration, quota errors, timeouts
  - Output format inconsistencies (sometimes LLM adds quotes, emojis, extra whitespace, etc.)

---

### 2.3 Customer Retrieval & Cleanup

**Overview:**  
Fetches all customers from WooCommerce, then cleans/normalizes their records (despite the node name referencing Shopify), then iterates over customers for WhatsApp verification/sending.

**Nodes Involved:**
- Get All Customer records from the WooCommerce store
- Clean Customer Data In Shopify Store
- Loop Over Customer

#### Node: Get All Customer records from the WooCommerce store
- **Type / role:** `n8n-nodes-base.wooCommerce` — WooCommerce API request to list customers.
- **Configuration choices:** Not visible. Typically:
  - Resource: Customers
  - Operation: Get Many / List
  - Pagination handling (important if store has many customers)
- **Input/Output:** From OpenAI node → outputs customers → `Clean Customer Data In Shopify Store`.
- **Version notes:** Type version `1`.
- **Edge cases / failures:**
  - Missing pagination → only first page of customers retrieved
  - Large customer count → long runs/timeouts
  - Credential/API permission errors
  - Rate limiting from WooCommerce host

#### Node: Clean Customer Data In Shopify Store
- **Type / role:** `n8n-nodes-base.code` — Data cleaning/normalization for customers (name suggests Shopify but input is WooCommerce).
- **Configuration choices:** Not provided. Likely tasks:
  - Normalize phone numbers to E.164
  - Remove customers without phone numbers
  - Extract fields (name, phone, product info, campaign copy)
- **Input/Output:** From WooCommerce customers → to `Loop Over Customer`.
- **Version notes:** Code node type version `2`.
- **Edge cases:**
  - Phone formats vary by country; normalization can be wrong if country missing
  - Customers without phone → should be filtered/logged (not shown explicitly)
  - If you output a single item with an array of customers, SplitInBatches may not iterate as intended unless you itemize customers

#### Node: Loop Over Customer
- **Type / role:** `n8n-nodes-base.splitInBatches` — Iterates through customers (commonly batch size 1).
- **Configuration choices:** Not visible.
- **Connections:**
  - Its **“continue” output (index 1)** goes to `Rapiwa (verify whatsapp number)`
  - First output is unused (similar wiring concern as earlier)
- **Version notes:** `3`.
- **Edge cases:**
  - Miswiring can cause loop not to execute as expected
  - Batch size impacts throughput and WhatsApp rate limiting

---

### 2.4 WhatsApp Verification, Sending & Logging

**Overview:**  
Verifies each customer number with Rapiwa, routes based on verification, sends message if verified, and logs results in separate Google Sheets. A Wait node is used after logging to pace executions.

**Nodes Involved:**
- Rapiwa (verify whatsapp number)
- If
- Rapiwa (send whatsapp message)
- Save data in Sheet Verified & Sent1
- Save data Sheet Unverified & Not sent1
- Wait1

#### Node: Rapiwa (verify whatsapp number)
- **Type / role:** `n8n-nodes-rapiwa.rapiwa` — Number verification / WhatsApp availability check.
- **Configuration choices:** Not visible; likely an operation like “validate number” or “check WhatsApp registration”.
- **Input/Output:** From customer loop → to `If`.
- **Failures:**
  - Auth/credential errors
  - Invalid phone number format
  - Provider latency/timeouts/rate limits

#### Node: If
- **Type / role:** `n8n-nodes-base.if` — Branches depending on verification result.
- **Configuration choices:** Not visible; expected to inspect Rapiwa verification response, e.g. `$json.verified === true`.
- **Connections:**
  - **True:** → `Rapiwa (send whatsapp message)`
  - **False:** → `Save data Sheet Unverified & Not sent1`
- **Version notes:** `2.2`.
- **Edge cases:**
  - Verification response shape changes → expression breaks
  - Truthy/falsey mismatches (e.g., `"true"` string)

#### Node: Rapiwa (send whatsapp message)
- **Type / role:** `n8n-nodes-rapiwa.rapiwa` — Sends WhatsApp message (likely including product short description and maybe image link).
- **Configuration choices:** Not visible. Typically requires:
  - To phone number (from customer item)
  - Message body (from OpenAI output + product data)
  - Possibly media/image URL(s)
- **Input/Output:** From If true → to `Save data in Sheet Verified & Sent1`.
- **Failure modes:**
  - Sending blocked by template policy (if using template-based sending)
  - Rate limits
  - Media URL invalid/unreachable
  - Message too long

#### Node: Save data in Sheet Verified & Sent1
- **Type / role:** `n8n-nodes-base.googleSheets` — Logs successful sends to a Google Sheet tab/range.
- **Configuration choices:** Not visible; likely “Append” operation with mapped columns (customer, number, product, timestamp, status).
- **Input/Output:** From send node → to `Wait1`.
- **Version notes:** `4.6`.
- **Failure modes:**
  - OAuth token expired / wrong Google account
  - Spreadsheet ID/tab missing
  - Column mismatch (n8n expects headers)
  - Google API rate limits

#### Node: Save data Sheet Unverified & Not sent1
- **Type / role:** `n8n-nodes-base.googleSheets` — Logs failures/unverified numbers to a different sheet/tab.
- **Configuration choices:** Not visible; likely “Append” with reason/status.
- **Input/Output:** From If false → to `Wait1`.
- **Version notes:** `4.6`.
- **Failure modes:** Same as other Google Sheets node.

#### Node: Wait1
- **Type / role:** `n8n-nodes-base.wait` — Pauses execution (throttling / pacing).
- **Configuration choices:** Not visible; can be time-based wait or “wait for webhook resume”. Node includes a `webhookId`, which is typical for Wait nodes.
- **Input/Output:** From both Sheets nodes → no downstream nodes shown (loop is not explicitly closed back to SplitInBatches).
- **Version notes:** `1.1`.
- **Edge cases:**
  - If configured to “wait for webhook”, execution will pause indefinitely unless resumed
  - If intended for rate limiting, time duration must be set; otherwise it may not throttle

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note2 | Sticky Note | Comment/annotation | — | — |  |
| Code (image link detect) | Code | Extract image links from WooCommerce product payload | WooCommerce Trigger | IF – Keeps only valid image files |  |
| IF – Keeps only valid image files | IF | Filter for valid image file links; branch if none | Code (image link detect) | Loop Over Image Link; Rapiwa (WhatsApp Notify No images found for the product) |  |
| Loop Over Image Link | SplitInBatches | Iterate through image links | IF – Keeps only valid image files | Create the product description (HTML) into a short |  |
| Rapiwa (WhatsApp Notify No images found for the product) | Rapiwa | Send admin WhatsApp alert when no images found | IF – Keeps only valid image files (false) | — |  |
| Create the product description (HTML) into a short | OpenAI (LangChain) | Generate short campaign copy from HTML description | Loop Over Image Link | Get All Customer records from the WooCommerce store |  |
| Get All Customer records from the WooCommerce store | WooCommerce | Fetch customer list from WooCommerce | Create the product description (HTML) into a short | Clean Customer Data In Shopify Store |  |
| Clean Customer Data In Shopify Store | Code | Normalize/clean customer records (phones, fields) | Get All Customer records from the WooCommerce store | Loop Over Customer |  |
| Loop Over Customer | SplitInBatches | Iterate through customers | Clean Customer Data In Shopify Store | Rapiwa (verify whatsapp number) |  |
| Rapiwa (verify whatsapp number) | Rapiwa | Verify if phone number is WhatsApp-capable | Loop Over Customer | If |  |
| If | IF | Route verified vs unverified | Rapiwa (verify whatsapp number) | Rapiwa (send whatsapp message); Save data Sheet Unverified & Not sent1 |  |
| Rapiwa (send whatsapp message) | Rapiwa | Send WhatsApp campaign message | If (true) | Save data in Sheet Verified & Sent1 |  |
| Save data in Sheet Verified & Sent1 | Google Sheets | Log successful sends | Rapiwa (send whatsapp message) | Wait1 |  |
| Save data Sheet Unverified & Not sent1 | Google Sheets | Log unverified / not sent | If (false) | Wait1 |  |
| Wait1 | Wait | Throttle/pause between operations | Save data in Sheet Verified & Sent1; Save data Sheet Unverified & Not sent1 | — |  |
| Sticky Note5 | Sticky Note | Comment/annotation | — | — |  |
| Sticky Note1 | Sticky Note | Comment/annotation | — | — |  |
| Sticky Note | Sticky Note | Comment/annotation | — | — |  |

> Note: All sticky notes have empty content in this JSON, so the “Sticky Note” column is blank.

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name it: *Automated WooCommerce New Product Campaigns for Customers* (or your preferred name).
- Ensure n8n has credentials set up for **WooCommerce**, **Google Sheets**, **OpenAI**, and **Rapiwa**.

2) **Add “WooCommerce Trigger”**
- Node type: **WooCommerce Trigger**
- Configure:
  - Connect WooCommerce credentials (API keys or OAuth depending on your setup)
  - Select the event corresponding to **new product created/published** (exact option depends on node UI)
- This becomes the workflow entry.

3) **Add “Code (image link detect)”**
- Node type: **Code**
- Purpose: extract image URLs from the trigger payload.
- Implement logic to output **one item per image** (recommended for later batching), e.g.:
  - Read product images array from `$json`
  - For each image, return an item like `{ imageUrl: 'https://...' }`
- Connect: **WooCommerce Trigger → Code (image link detect)**

4) **Add “IF – Keeps only valid image files”**
- Node type: **IF**
- Configure condition(s) such as:
  - `{{$json.imageUrl}}` matches regex for `\.(jpg|jpeg|png|webp|gif)(\?.*)?$`
- Connect: **Code (image link detect) → IF – Keeps only valid image files**

5) **Add “Rapiwa (WhatsApp Notify No images found for the product)”**
- Node type: **Rapiwa**
- Operation: send WhatsApp message (to an admin number).
- Message content example: “No valid images found for product: {{$json.name || $json.product?.name}}”
- Connect **IF false output → this Rapiwa node**

6) **Add “Loop Over Image Link”**
- Node type: **Split In Batches**
- Set batch size to **1** (recommended).
- Connect **IF true output → Loop Over Image Link**
- Recommended wiring pattern in n8n:
  - Use the SplitInBatches first output to process the batch
  - Then connect the last node of that sub-chain back to SplitInBatches to continue
- (In the provided JSON, the “continue” output is used; consider correcting this when rebuilding.)

7) **Add “Create the product description (HTML) into a short”**
- Node type: **OpenAI (LangChain)**
- Credentials: OpenAI API key.
- Configure:
  - Provide the product HTML description as input (from the trigger payload).
  - Prompt to produce WhatsApp-friendly short copy (e.g., 1–3 lines, include price/CTA if present).
- Connect from **Loop Over Image Link → OpenAI node**.

8) **Add “Get All Customer records from the WooCommerce store”**
- Node type: **WooCommerce**
- Resource: **Customer**
- Operation: **Get Many/List**
- Enable pagination / “Return All” if available.
- Connect: **OpenAI node → WooCommerce (Get customers)**

9) **Add “Clean Customer Data In Shopify Store” (rename optional)**
- Node type: **Code**
- Implement:
  - Extract phone numbers, normalize to E.164 (require country context if possible)
  - Filter out empty phones
  - Carry forward campaign text, product name, image URL(s)
  - Output **one item per customer**
- Connect: **Get customers → Clean code node**

10) **Add “Loop Over Customer”**
- Node type: **Split In Batches**
- Batch size **1** (recommended).
- Connect: **Clean code node → Loop Over Customer**
- As with the image loop, use a proper “loop back” pattern if you need continuous iteration with throttling.

11) **Add “Rapiwa (verify whatsapp number)”**
- Node type: **Rapiwa**
- Operation: verify/check WhatsApp number.
- Map the phone field from your cleaned customer item.
- Connect: **Loop Over Customer → Verify node**

12) **Add “If” (verification router)**
- Node type: **IF**
- Condition: check verification result field returned by Rapiwa (example: `{{$json.isWhatsapp === true}}`).
- Connect: **Verify node → If**

13) **Add “Rapiwa (send whatsapp message)”**
- Node type: **Rapiwa**
- Operation: send WhatsApp message (and optionally media).
- Map:
  - To: customer phone
  - Body: OpenAI short description + product link
  - Media: image URL if supported
- Connect: **If true → Send message**

14) **Add two Google Sheets logging nodes**
- Node type: **Google Sheets**
- Credentials: Google OAuth2.
- Node A: **Save data in Sheet Verified & Sent1**
  - Operation: Append row
  - Sheet/tab: “Verified & Sent”
  - Columns: phone, customer name/id, product id/name, timestamp, status, message id, etc.
  - Connect: **Send message → Verified sheet**
- Node B: **Save data Sheet Unverified & Not sent1**
  - Operation: Append row
  - Sheet/tab: “Unverified & Not sent”
  - Columns: phone, customer, reason/status, timestamp
  - Connect: **If false → Unverified sheet**

15) **Add “Wait1”**
- Node type: **Wait**
- Configure a delay (e.g., 1–3 seconds) if you want rate limiting, **or** configure “resume via webhook” if you want manual control (only if you understand the implications).
- Connect:
  - **Verified sheet → Wait1**
  - **Unverified sheet → Wait1**
- If you want a real customer loop throttle, connect **Wait1 back to “Loop Over Customer”** (continue output) to proceed to the next batch.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Sticky notes exist but contain no text in the provided workflow JSON. | Applies to Sticky Note, Sticky Note1, Sticky Note2, Sticky Note5 |

If you want, I can propose the exact missing configurations (IF conditions, Code node scripts, OpenAI prompt, and the correct SplitInBatches looping wiring) based on your WooCommerce payload sample and your desired WhatsApp message format.
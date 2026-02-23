Send Shopify new product announcements via WhatsApp using Google Sheets and GPT-4.1 mini

https://n8nworkflows.xyz/workflows/send-shopify-new-product-announcements-via-whatsapp-using-google-sheets-and-gpt-4-1-mini-13415


# Send Shopify new product announcements via WhatsApp using Google Sheets and GPT-4.1 mini

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow listens for **new Shopify product events**, prepares promotional content (including an AI-generated HTML description), looks up **pending recipient rows in Google Sheets**, **verifies WhatsApp numbers via Rapiwa**, and sends WhatsApp announcements. It then updates Google Sheets with “sent/not sent” statuses and throttles sending with a short wait.

**Target use cases:**
- Automatically announce newly created Shopify products to a WhatsApp contact list stored in Google Sheets
- Reduce manual copywriting by generating a product description using GPT-4.1 mini (via OpenAI Chat Model node)
- Prevent failed sends by verifying WhatsApp numbers before sending

### Logical blocks (by dependency)
1.1 **Shopify Event Reception & Image Pre-check**  
1.2 **Per-Image Loop & Product URL Construction**  
1.3 **AI Content Generation (HTML description)**  
1.4 **Fetch Pending Recipients from Google Sheets + Throttle/Limit**  
1.5 **Recipient Loop → Number Verification → Conditional Send**  
1.6 **Status Updates (Sent vs Not Sent) + Rate Control Wait**

---

## 2. Block-by-Block Analysis

### 2.1 Shopify Event Reception & Image Pre-check

**Overview:**  
Triggers on Shopify events, then runs a code-based “image quality detection” step and branches depending on whether valid image links exist.

**Nodes involved:**
- Shopify Trigger
- File image quality detect switch (Code)
- If (Check Valid Image Links)
- Rapiwa (WhatsApp Notify No images found for the product)

#### Node: **Shopify Trigger**
- **Type / role:** `n8n-nodes-base.shopifyTrigger` — entry point via Shopify webhook trigger.
- **Configuration (interpreted):** Parameters not shown in JSON (empty), so the event/topic selection and authentication must be configured in the UI (commonly “Product created/updated”).
- **Connections:**  
  - Output → **File image quality detect switch**
- **Version-specific notes:** TypeVersion `1` (older trigger version; UI options may differ from newer Shopify nodes).
- **Failure/edge cases:**
  - Missing/invalid Shopify credentials
  - Webhook not registered/disabled in Shopify
  - Shopify sends product payloads without images (common for draft products)

#### Node: **File image quality detect switch**
- **Type / role:** `n8n-nodes-base.code` — transforms/inspects incoming product data (likely evaluates images).
- **Configuration choices:** Not present in JSON (empty parameters), meaning the actual JS code is missing from this export; the workflow depends on custom logic here.
- **Connections:**  
  - Input ← Shopify Trigger  
  - Output → **If (Check Valid Image Links)**
- **Failure/edge cases:**
  - If code is empty or errors, the workflow stops here.
  - If it expects certain Shopify fields (e.g., `images`, `image.src`) and they’re absent, it may throw.

#### Node: **If (Check Valid Image Links)**
- **Type / role:** `n8n-nodes-base.if` — routes execution based on presence/validity of image links.
- **Configuration choices:** Not shown (empty). It should evaluate the output of the Code node (e.g., `{{$json.validImages.length > 0}}`).
- **Connections:**
  - **True** → **Loop Over Items** (continue with images/products)
  - **False** → **Rapiwa (WhatsApp Notify No images found for the product)**
- **Failure/edge cases:**
  - Expression errors if expected fields are missing (e.g., `validImages` not defined)

#### Node: **Rapiwa (WhatsApp Notify No images found for the product)**
- **Type / role:** `n8n-nodes-rapiwa.rapiwa` — sends a WhatsApp notification if no images were found/valid.
- **Configuration choices:** Not shown (empty). Typically includes: recipient number(s), message text, and sender/channel configuration.
- **Connections:** Terminal branch (no outgoing connections in JSON).
- **Failure/edge cases:**
  - Rapiwa credential/API key missing
  - WhatsApp template/session rules not satisfied (depending on provider/account)
  - Invalid recipient number formatting

---

### 2.2 Per-Image Loop & Product URL Construction

**Overview:**  
Iterates over items (likely images or product variants) and constructs a product URL for downstream content generation.

**Nodes involved:**
- Loop Over Items
- Create Product URL

#### Node: **Loop Over Items**
- **Type / role:** `n8n-nodes-base.splitInBatches` — batching/looping mechanism.
- **Configuration choices:** Not shown; defaults commonly batch size = 1.
- **Connections:**
  - Uses **Output 1 (Next batch)** → (not connected)
  - Uses **Output 2 (Done)** → **Create Product URL**
  - This connection pattern suggests the node is being used in a non-standard way (normally “Next batch” is the main loop path).
- **Failure/edge cases:**
  - If batch size is misconfigured, you may get no iterations or unintended grouping.
  - If no input items arrive (e.g., no images), done-path might execute immediately.

#### Node: **Create Product URL**
- **Type / role:** `n8n-nodes-base.set` — creates/sets a product URL field for later use.
- **Configuration choices:** Not shown. Common pattern:
  - `productUrl = https://{shop-domain}/products/{{$json.handle}}`
- **Connections:**
  - Output → **Create the product’s HTML description**
- **Failure/edge cases:**
  - Missing product `handle` or store domain
  - If Shopify is multi-market, URL might need locale paths

---

### 2.3 AI Content Generation (HTML description)

**Overview:**  
Uses an OpenAI chat model (GPT-4.1 mini) within a LangChain Agent node to generate an HTML description for the product.

**Nodes involved:**
- OpenAI Chat Model
- Create the product’s HTML description

#### Node: **OpenAI Chat Model**
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — defines the LLM used by the agent.
- **Configuration choices:** Not shown in JSON. The workflow title indicates **GPT-4.1 mini**.
- **Connections:**
  - Output (AI language model) → **Create the product’s HTML description** (as `ai_languageModel`)
- **Version-specific notes:** TypeVersion `1.2`.
- **Failure/edge cases:**
  - Missing OpenAI credentials
  - Model not available in the account/region
  - Token limits if product text/images metadata is large

#### Node: **Create the product’s HTML description**
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — prompts the model and returns generated content.
- **Configuration choices:** Not shown. Typically includes:
  - System/instructions to produce HTML
  - Inputs from Shopify (title, vendor, price, tags, images, URL)
- **Connections:**
  - Input ← **Create Product URL**
  - Uses LLM from **OpenAI Chat Model**
  - Output → **Fetch All Pending Queries for Messaging**
- **Failure/edge cases:**
  - Prompt expects fields that aren’t present
  - HTML output might contain unsupported formatting for WhatsApp (WhatsApp supports limited formatting; raw HTML may need conversion)

---

### 2.4 Fetch Pending Recipients from Google Sheets + Throttle/Limit

**Overview:**  
Pulls rows from Google Sheets representing pending recipients to message, then limits the volume before entering the recipient loop.

**Nodes involved:**
- Fetch All Pending Queries for Messaging
- Limit

#### Node: **Fetch All Pending Queries for Messaging**
- **Type / role:** `n8n-nodes-base.googleSheets` — reads rows from a spreadsheet.
- **Configuration choices:** Not shown. Likely:
  - Operation: Read/Get Many
  - Filter: `status = pending` (or equivalent)
- **Connections:**
  - Output → **Limit**
- **Version-specific notes:** TypeVersion `4.6` (modern Sheets node; supports structured operations).
- **Failure/edge cases:**
  - Google auth expired (OAuth2)
  - Sheet/tab name changed
  - Filtering not applied properly → sends to wrong recipients

#### Node: **Limit**
- **Type / role:** `n8n-nodes-base.limit` — caps the number of rows processed per run.
- **Configuration choices:** Not shown (e.g., max items 10/50).
- **Connections:**
  - Output → **Loop Over Items1**
- **Failure/edge cases:**
  - If set too low, many recipients remain pending.
  - If set too high, you may hit WhatsApp provider rate limits.

---

### 2.5 Recipient Loop → Number Verification → Conditional Send

**Overview:**  
Loops over the limited recipient rows, verifies each WhatsApp number using Rapiwa, and sends only if verified; otherwise marks not sent.

**Nodes involved:**
- Loop Over Items1
- Verify WhatsApp Numbers Using Rapiwa
- If (Check Verify & Unverified Number)
- Rapiwa (Sent WhatsApp Message)

#### Node: **Loop Over Items1**
- **Type / role:** `n8n-nodes-base.splitInBatches` — iterates through recipient rows.
- **Configuration choices:** Not shown; `executeOnce: false` indicates normal looping.
- **Connections:**
  - Output 2 (Done) → **Verify WhatsApp Numbers Using Rapiwa** (again, unusual: typically output 1 drives the loop)
  - Also receives input from:
    - **Limit**
    - **Wait (5s)** (to continue next batch after waiting)
- **Failure/edge cases:**
  - Incorrect output wiring can cause only “done” to execute, skipping per-row logic, depending on configuration.
  - If batch size > 1, verification/sending may not align with expected per-row processing.

#### Node: **Verify WhatsApp Numbers Using Rapiwa**
- **Type / role:** `n8n-nodes-rapiwa.rapiwa` — calls Rapiwa API to validate WhatsApp numbers.
- **Configuration choices:** Not shown. Typically passes phone number from the current sheet row.
- **Connections:**
  - Output → **If (Check Verify & Unverified Number)**
- **Failure/edge cases:**
  - E.164 formatting required (e.g., `+15551234567`)
  - Provider may return ambiguous statuses; the IF condition must match the real response schema

#### Node: **If (Check Verify & Unverified Number)**
- **Type / role:** `n8n-nodes-base.if` — branches based on verification result.
- **Configuration choices:** Not shown. Usually checks something like `{{$json.verified === true}}`.
- **Connections:**
  - **True** → **Rapiwa (Sent WhatsApp Message)**
  - **False** → **Update Status of Rows Unverified & Not Sent**
- **Failure/edge cases:**
  - If verification response fields differ, all numbers may be treated as unverified.

#### Node: **Rapiwa (Sent WhatsApp Message)**
- **Type / role:** `n8n-nodes-rapiwa.rapiwa` — sends WhatsApp message to verified numbers.
- **Configuration choices:** Not shown. Likely message body uses:
  - Product title
  - Product URL
  - AI-generated description
  - Image link(s)
- **Connections:**
  - Output → **Update Status of Rows Verified & Sent**
- **Failure/edge cases:**
  - Media link must be publicly accessible (HTTPS)
  - Rate limits, template requirements, session windows
  - Message length constraints

---

### 2.6 Status Updates (Sent vs Not Sent) + Rate Control Wait

**Overview:**  
Updates the Google Sheets row status depending on send outcome, waits 5 seconds, then continues the recipient loop.

**Nodes involved:**
- Update Status of Rows Verified & Sent
- Update Status of Rows Unverified & Not Sent
- Wait (5s)

#### Node: **Update Status of Rows Verified & Sent**
- **Type / role:** `n8n-nodes-base.googleSheets` — marks row as sent/verified.
- **Configuration choices:** Not shown. Typically an “Update” operation using row ID / row number from the fetched items.
- **Connections:**
  - Output → **Wait (5s)**
- **Failure/edge cases:**
  - Wrong row mapping updates the wrong recipient
  - Concurrency: parallel runs can overwrite statuses if not locked

#### Node: **Update Status of Rows Unverified & Not Sent**
- **Type / role:** `n8n-nodes-base.googleSheets` — marks row as not sent/unverified.
- **Configuration choices:** Not shown.
- **Connections:**
  - Output → **Wait (5s)**
- **Failure/edge cases:**
  - Same as above; also ensure you keep a reason column for debugging.

#### Node: **Wait (5s)**
- **Type / role:** `n8n-nodes-base.wait` — pauses between sends to reduce rate-limit risk.
- **Configuration choices:** Not shown; name implies 5 seconds.
- **Connections:**
  - Output → **Loop Over Items1** (to proceed with the next batch/recipient)
- **Failure/edge cases:**
  - If configured as “Wait for webhook” instead of “time interval”, it will stall indefinitely. Ensure it’s time-based.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Shopify Trigger | shopifyTrigger | Entry point from Shopify webhook | — | File image quality detect switch |  |
| File image quality detect switch | code | Custom preprocessing of product/images | Shopify Trigger | If (Check Valid Image Links) |  |
| If (Check Valid Image Links) | if | Branch if product has valid images | File image quality detect switch | Loop Over Items; Rapiwa (WhatsApp Notify No images found for the product) |  |
| Loop Over Items | splitInBatches | Iterate/batch over items (likely images) | If (Check Valid Image Links) | Create Product URL (via “done” output) |  |
| Create Product URL | set | Build product URL field | Loop Over Items | Create the product’s HTML description |  |
| OpenAI Chat Model | lmChatOpenAi | Provides GPT-4.1 mini chat model to agent | — | Create the product’s HTML description (ai_languageModel) |  |
| Create the product’s HTML description | langchain.agent | Generate HTML description for product | Create Product URL; OpenAI Chat Model | Fetch All Pending Queries for Messaging |  |
| Fetch All Pending Queries for Messaging | googleSheets | Read pending recipient rows | Create the product’s HTML description | Limit |  |
| Limit | limit | Cap number of recipients per run | Fetch All Pending Queries for Messaging | Loop Over Items1 |  |
| Loop Over Items1 | splitInBatches | Iterate over recipients | Limit; Wait (5s) | Verify WhatsApp Numbers Using Rapiwa (via “done” output) |  |
| Verify WhatsApp Numbers Using Rapiwa | rapiwa | Validate WhatsApp number | Loop Over Items1 | If (Check Verify & Unverified Number) |  |
| If (Check Verify & Unverified Number) | if | Branch on verification result | Verify WhatsApp Numbers Using Rapiwa | Rapiwa (Sent WhatsApp Message); Update Status of Rows Unverified & Not Sent |  |
| Rapiwa (Sent WhatsApp Message) | rapiwa | Send WhatsApp promo message | If (Check Verify & Unverified Number) | Update Status of Rows Verified & Sent |  |
| Update Status of Rows Verified & Sent | googleSheets | Mark row sent/verified | Rapiwa (Sent WhatsApp Message) | Wait (5s) |  |
| Update Status of Rows Unverified & Not Sent | googleSheets | Mark row not sent/unverified | If (Check Verify & Unverified Number) | Wait (5s) |  |
| Wait (5s) | wait | Throttle between sends | Update Status of Rows Verified & Sent; Update Status of Rows Unverified & Not Sent | Loop Over Items1 |  |
| Rapiwa (WhatsApp Notify No images found for the product) | rapiwa | Alert when product has no images | If (Check Valid Image Links) | — |  |
| Sticky Note | stickyNote | Comment container (empty) | — | — |  |
| Sticky Note1 | stickyNote | Comment container (empty) | — | — |  |
| Sticky Note2 | stickyNote | Comment container (empty) | — | — |  |
| Sticky Note12 | stickyNote | Comment container (empty) | — | — |  |
| Sticky Note13 | stickyNote | Comment container (empty) | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1) **Create workflow**
   1. Create a new workflow named: *Automate Shopify New Product Promotions via WhatsApp from Sheets*.
   2. (Optional) Set workflow title/description as provided.

2) **Add Shopify trigger**
   1. Add node: **Shopify Trigger**.
   2. Configure Shopify credentials (Admin API access).
   3. Select the trigger event/topic relevant to “new product” (commonly Product Create).
   4. Save and (later) activate to register webhook.

3) **Add product/image preprocessing**
   1. Add node: **Code** named *File image quality detect switch*.
   2. Implement JS to extract product images, filter valid URLs, and output fields used by later nodes (at minimum: a list/flag used by the IF node).
   3. Connect: **Shopify Trigger → Code**.

4) **Add image-link validation branch**
   1. Add node: **If** named *If (Check Valid Image Links)*.
   2. Condition: check that one or more valid image URLs exist (based on Code output).
   3. Connect: **Code → If**.
   4. On **False**, add **Rapiwa** node named *Rapiwa (WhatsApp Notify No images found for the product)* and configure:
      - Credentials: Rapiwa API key/account
      - Recipient: your internal alert number/team number
      - Message text: “No images found for product {{product title/id}} …”
   5. Connect: **If (False) → Notify node**.

5) **Loop over items (images or product items)**
   1. Add node: **Split In Batches** named *Loop Over Items*.
   2. Set batch size (commonly 1).
   3. Connect: **If (True) → Loop Over Items**.
   4. Ensure the loop output you use actually processes each item (recommended: use Output 1 for per-batch processing; Output 2 for completion). Adjust connections accordingly.

6) **Create product URL**
   1. Add node: **Set** named *Create Product URL*.
   2. Add a field like `productUrl` built from your shop domain + product handle.
   3. Connect: **Loop Over Items → Create Product URL**.

7) **Configure OpenAI model + agent**
   1. Add node: **OpenAI Chat Model** (LangChain) named *OpenAI Chat Model*.
   2. Select model: **GPT-4.1 mini** (or closest available in your n8n/OpenAI account).
   3. Configure OpenAI credentials.
   4. Add node: **AI Agent** named *Create the product’s HTML description*.
   5. In the agent, craft instructions to generate an HTML description using product fields and `productUrl`.
   6. Connect:
      - **Create Product URL → Agent**
      - **OpenAI Chat Model (ai_languageModel) → Agent**

8) **Read pending recipients from Google Sheets**
   1. Add node: **Google Sheets** named *Fetch All Pending Queries for Messaging*.
   2. Configure Google OAuth2 credentials.
   3. Select Spreadsheet + Sheet/tab.
   4. Operation: read/get many rows; apply filter to only fetch “pending” rows (status column).
   5. Connect: **Agent → Google Sheets (Fetch)**.

9) **Limit recipients**
   1. Add node: **Limit** named *Limit*.
   2. Set maximum items per run (e.g., 10–50).
   3. Connect: **Fetch → Limit**.

10) **Loop over recipients**
   1. Add node: **Split In Batches** named *Loop Over Items1*.
   2. Batch size: 1 (recommended).
   3. Connect: **Limit → Loop Over Items1**.

11) **Verify WhatsApp number**
   1. Add node: **Rapiwa** named *Verify WhatsApp Numbers Using Rapiwa*.
   2. Configure operation for number verification (per Rapiwa node capabilities).
   3. Map input phone number from the current sheet row.
   4. Connect: **Loop Over Items1 → Verify**.

12) **Branch verified vs unverified**
   1. Add node: **If** named *If (Check Verify & Unverified Number)*.
   2. Condition: match Rapiwa verification response (e.g., verified boolean/status).
   3. Connect: **Verify → If**.

13) **Send WhatsApp message (verified path)**
   1. Add node: **Rapiwa** named *Rapiwa (Sent WhatsApp Message)*.
   2. Configure send-message operation:
      - Recipient: current row phone
      - Message body: include product name, `productUrl`, and AI output (ensure WhatsApp-compatible formatting; HTML may need plain text conversion).
      - Media: include image URL if supported.
   3. Connect: **If (True) → Send**.

14) **Update Google Sheets status**
   - Verified & sent:
     1. Add **Google Sheets** node named *Update Status of Rows Verified & Sent*.
     2. Operation: Update row.
     3. Set `status = sent` and store timestamp/messageId if available.
     4. Connect: **Send → Update Verified**.
   - Unverified & not sent:
     1. Add **Google Sheets** node named *Update Status of Rows Unverified & Not Sent*.
     2. Operation: Update row.
     3. Set `status = unverified` (or `not_sent`) and store reason.
     4. Connect: **If (False) → Update Unverified**.

15) **Wait and continue loop**
   1. Add node: **Wait** named *Wait (5s)*.
   2. Configure to wait a fixed duration: 5 seconds.
   3. Connect:
      - **Update Verified → Wait**
      - **Update Unverified → Wait**
      - **Wait → Loop Over Items1** (continue to next recipient)

16) **Finalize**
   1. Test with a Shopify product creation event.
   2. Confirm:
      - Images pass validation
      - Sheet rows are correctly filtered and updated
      - Verified numbers send successfully
      - Unverified numbers are marked correctly
   3. Activate workflow.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Sticky notes exist but contain no text in this workflow export. | No links/comments were provided in sticky notes. |
| Several nodes show empty `parameters` in the JSON export (notably Code/If/Set/Sheets/Rapiwa/Agent). The functional logic depends on UI configuration and/or missing code. | When recreating, you must define: image validation logic, URL construction, Sheets filters, WhatsApp message templates, and verification checks. |
| The Split In Batches nodes are connected in a way that suggests “done” outputs are being used for processing. Review wiring to ensure per-item iteration works as intended. | Common fix: drive processing from Output 1 and loop using the node’s “Continue” pattern. |
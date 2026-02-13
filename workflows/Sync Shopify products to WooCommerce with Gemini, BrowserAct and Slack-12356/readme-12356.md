Sync Shopify products to WooCommerce with Gemini, BrowserAct and Slack

https://n8nworkflows.xyz/workflows/sync-shopify-products-to-woocommerce-with-gemini--browseract-and-slack-12356


# Sync Shopify products to WooCommerce with Gemini, BrowserAct and Slack

## Disclaimer  
Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

# Sync Shopify products to WooCommerce with Gemini, BrowserAct and Slack — Reference Document

## 1. Workflow Overview

**Purpose:**  
This workflow synchronizes products from **Shopify** to **WooCommerce**. For each Shopify product, it uses **Gemini (via n8n LangChain nodes)** plus an optional **BrowserAct enrichment workflow** to generate clean WooCommerce-ready product data (title, SKU, price, HTML description, images). It prevents duplicates by checking WooCommerce for an existing SKU. It posts status notifications to **Slack**, including error alerts when product creation fails.

**Primary use cases:**
- Bulk migration or ongoing sync of products from Shopify to WooCommerce
- AI-enhanced product listings (structured HTML descriptions, SKU generation, image metadata)
- Duplicate avoidance using SKU checks
- Team visibility via Slack notifications

### Logical Blocks
1.1 **Trigger & Source Fetch (Shopify)**  
Manual trigger → fetch all Shopify products → iterate product-by-product.

1.2 **AI Classification (Gemini) → Decide Enrichment Target**  
Gemini analyzes product title and outputs structured JSON containing `Products` and `BestSite` to guide BrowserAct enrichment.

1.3 **BrowserAct Enrichment (External Workflow Execution)**  
Runs a BrowserAct workflow (by ID) to scrape/collect product information from the selected “BestSite”.

1.4 **AI Data Modeling (Gemini) → WooCommerce-Ready Product JSON**  
Gemini transforms enriched raw data into a strict structured object (title, SKU, price, HTML description, image list).

1.5 **WooCommerce Duplicate Check (SKU)**  
Queries WooCommerce for products with the same SKU.

1.6 **Conditional Create + Slack Notifications**  
If missing → create product in WooCommerce.  
If found → skip creation and continue.  
On create error → send Slack error message, then continue loop.  
When loop ends → send Slack completion message.

---

## 2. Block-by-Block Analysis

### 2.1 Trigger & Shopify Fetch + Iteration
**Overview:**  
Starts the workflow manually, pulls all Shopify products, then processes them one at a time using batching/looping.

**Nodes involved:**
- Manual Execution
- Get Shopify Products
- Loop Over Items
- Notify Slack Team of Completion

#### Node: Manual Execution
- **Type / Role:** Manual Trigger (`n8n-nodes-base.manualTrigger`) — entry point for testing/runs.
- **Configuration choices:** No parameters; user starts run manually.
- **Connections:**  
  - Output → **Get Shopify Products**
- **Failure modes / edge cases:** None (except workflow not started).

#### Node: Get Shopify Products
- **Type / Role:** Shopify (`n8n-nodes-base.shopify`) — retrieves products from Shopify.
- **Configuration choices:**
  - **Resource:** Product
  - **Operation:** Get All
  - **Return All:** true (pulls the full catalog)
  - **Auth:** Access Token
- **Credentials:** `Shopify Access Token account`
- **Connections:**  
  - Input ← Manual Execution  
  - Output → **Loop Over Items**
- **Edge cases / failures:**
  - Invalid/expired token → 401/403
  - Large catalogs may increase runtime/memory; Shopify API rate limiting could appear.

#### Node: Loop Over Items
- **Type / Role:** Split In Batches (`n8n-nodes-base.splitInBatches`) — iterates through products.
- **Configuration choices:** Default options (batch size not explicitly set in JSON; n8n defaults apply).
- **Connections:**
  - Input ← Get Shopify Products
  - Output (main, branch 1) → **Analyze the Products** (per-item processing)
  - Output (main, branch 0) → **Notify Slack Team of Completion** (when no items left / loop completion path)
  - Also receives inputs back from: **Check for Product availability**, **Import Product to WooCommerce**, **Send Error** to continue looping.
- **Edge cases / failures:**
  - If batch size is too large, downstream nodes may be slower; if too small, overhead increases.
  - Miswiring loop-back paths can cause infinite loops (this workflow correctly loops back only after create/skip).

#### Node: Notify Slack Team of Completion
- **Type / Role:** Slack (`n8n-nodes-base.slack`) — posts completion message after loop ends.
- **Configuration choices:**
  - **Message:** `Product sync from Shopify to WooCommerce is done.`
  - **Target:** Channel (ID `C09KLV9DJSX`, cached name `all-browseract-workflow-test`)
- **Credentials:** `Slack account 2`
- **Connections:**
  - Input ← Loop Over Items (completion path)
- **Edge cases / failures:**
  - Slack auth revoked/invalid → posting fails
  - Channel ID mismatch or bot not in channel → `channel_not_found` / permission errors

---

### 2.2 AI Classification & Enrichment Target Selection (Gemini + Structured Output)
**Overview:**  
For each Shopify product, Gemini analyzes the product title and outputs strict JSON containing a normalized product name and the best reference site (Amazon/Trustpilot/G2/Steam). A structured output parser enforces JSON validity.

**Nodes involved:**
- Google Gemini
- Structured Output
- Analyze the Products

#### Node: Google Gemini
- **Type / Role:** LangChain Chat Model (`@n8n/n8n-nodes-langchain.lmChatGoogleGemini`) — provides the LLM used by the agent/parser.
- **Configuration choices:**
  - **Model:** `models/gemini-2.5-pro`
- **Credentials:** `Google Gemini(PaLM) Api account`
- **Connections:**
  - AI Language Model → **Analyze the Products**
  - AI Language Model → **Structured Output**
- **Edge cases / failures:**
  - API key invalid/billing issues
  - Model availability changes or quota exhaustion
  - Latency/timeouts

#### Node: Structured Output
- **Type / Role:** Structured Output Parser (`@n8n/n8n-nodes-langchain.outputParserStructured`) — validates/repairs LLM output into JSON.
- **Configuration choices:**
  - **Auto-fix:** enabled (attempts to correct malformed JSON)
  - **Schema example:**  
    ```text
    {
      "Type": "Comparison",
      "Products": "Product",
      "BestSite": "Name of the site"
    }
    ```
  - Note: the agent system message expects `Type: "GetData"`, while the parser example shows `"Comparison"`. This inconsistency usually won’t break parsing but can cause type drift.
- **Connections:**
  - Output Parser → **Analyze the Products** (agent uses this parser)
  - Receives model from **Google Gemini**
- **Edge cases / failures:**
  - If the model outputs non-JSON repeatedly, auto-fix may still fail.
  - Schema mismatch could lead to missing keys (`BestSite`, `Products`).

#### Node: Analyze the Products
- **Type / Role:** LangChain Agent (`@n8n/n8n-nodes-langchain.agent`) — classifies product title into a structured object.
- **Configuration choices:**
  - **Input text:** `{{ $json.title }}`
  - **System message:** strict “Input Classification Engine” with rules mapping product type → `BestSite`:
    - Physical products → Amazon
    - Services/websites → Trustpilot
    - SaaS → G2
    - Video games → Steam
  - **Output constraints:** raw JSON only; `BestSite` single string.
  - **Has output parser:** true (uses **Structured Output**)
- **Connections:**
  - Input ← Loop Over Items
  - Output → **Search for Product Info**
  - Uses:
    - LLM: **Google Gemini**
    - Parser: **Structured Output**
- **Edge cases / failures:**
  - Shopify products without `title` field (unexpected) → empty classification text
  - Ambiguous items (e.g., “Gift Card”, “Membership”) may be misclassified
  - LLM hallucinations: wrong `Products` string may degrade BrowserAct results

---

### 2.3 BrowserAct Enrichment (External Workflow Invocation)
**Overview:**  
Calls a BrowserAct workflow to retrieve additional product data (specs, images, reviews, price, etc.) from the chosen site. This enrichment feeds the second AI step.

**Nodes involved:**
- Search for Product Info

#### Node: Search for Product Info
- **Type / Role:** BrowserAct Workflow Runner (`n8n-nodes-browseract.browserAct`) — executes a BrowserAct “WORKFLOW”.
- **Configuration choices:**
  - **Mode/Type:** WORKFLOW
  - **Timeout:** 7200 seconds (2 hours)
  - **Workflow ID:** `70444920944876283`
  - **Workflow inputs mapping:**
    - `input-Product` = `{{ $json.output.Products }}`
    - `input-Target_Site` = `{{ $json.output.BestSite }}`
  - Incognito mode: false
  - Mapping mode: defineBelow
- **Credentials:** `BrowserAct account`
- **Connections:**
  - Input ← Analyze the Products
  - Output → **Create Product Data**
- **Edge cases / failures:**
  - BrowserAct workflow missing/renamed or ID invalid → execution error
  - Scraping failures due to captchas, layout changes, blocked requests
  - Long runtime: even with 2h timeout, runs may time out for problematic pages
  - Output format drift: downstream AI assumes `{{$json.output.string}}` exists (see next block)

**Sub-workflow reference:**  
This node invokes an external BrowserAct workflow (not an n8n sub-workflow node). Reproduction requires having the BrowserAct workflow/template available and matching its expected output shape.

---

### 2.4 AI Product Data Creation (Gemini + Structured Output)
**Overview:**  
Transforms BrowserAct’s raw output into a WooCommerce-ready product object: `title`, generated `sku`, `price`, `html_body`, and `image_list` with at least two images. A structured parser enforces JSON shape.

**Nodes involved:**
- Google Gemini1
- Structured Output1
- Create Product Data

#### Node: Google Gemini1
- **Type / Role:** LangChain Chat Model — model provider for the second agent.
- **Configuration choices:**
  - **Model:** `models/gemini-2.5-pro`
- **Credentials:** `Google Gemini(PaLM) Api account`
- **Connections:**
  - AI Language Model → **Create Product Data**
  - AI Language Model → **Structured Output1**
- **Edge cases / failures:** same as other Gemini node (quota, auth, latency).

#### Node: Structured Output1
- **Type / Role:** Structured Output Parser — enforces the WooCommerce product JSON schema.
- **Configuration choices:**
  - **Auto-fix:** enabled
  - **Schema example includes:**
    - `title` (string)
    - `sku` (string)
    - `price` (string)
    - `html_body` (string, HTML)
    - `image_list` (array of `{url,name,alt}`)
- **Connections:**
  - Output Parser → **Create Product Data**
  - Receives model from **Google Gemini1**
- **Edge cases / failures:**
  - If the model outputs invalid HTML inside JSON without escaping, parser may fail
  - If BrowserAct output lacks `Images`, agent may invent URLs; later WooCommerce image upload may fail

#### Node: Create Product Data
- **Type / Role:** LangChain Agent — generates final product payload.
- **Configuration choices:**
  - **Input text:** `{{ $json.output.string }}`
    - Assumes BrowserAct returns `output.string` containing raw product info.
  - **System message:** “E-Commerce Data Architect and Senior Copywriter” rules:
    - Title from `Name`
    - Generate SKU from Brand/Model/specs/color (format like `BRAND-MODEL-SPEC-COLOR`)
    - Price from `Price`
    - Description as HTML with `<h2>`, `<p>`, `<ul>/<li>`
    - Parse Images string separated by `||` into list
    - Always output two images; duplicate if only one
  - **Has output parser:** true (uses Structured Output1)
- **Connections:**
  - Input ← Search for Product Info
  - Output → **Get WooCommerce Products**
- **Edge cases / failures:**
  - If BrowserAct does not output `output.string`, expression resolves to empty → weak or invalid result
  - SKU collisions possible (two different products generate same SKU)
  - Price formatting may not match WooCommerce expectation (should typically be numeric string without currency symbol depending on WooCommerce settings; this workflow uses “String (currency and value)”)

---

### 2.5 WooCommerce Duplicate Check (SKU)
**Overview:**  
Queries WooCommerce for products with the generated SKU, then routes based on whether a match exists.

**Nodes involved:**
- Get WooCommerce Products
- Check for Product availability

#### Node: Get WooCommerce Products
- **Type / Role:** WooCommerce (`n8n-nodes-base.wooCommerce`) — looks up products by SKU.
- **Configuration choices:**
  - **Operation:** Get All
  - **Return All:** true
  - **Filter/Option:** `sku = {{ $json.output.sku }}`
  - **alwaysOutputData:** true (important: even if nothing is found, it outputs an item so downstream logic can run)
- **Credentials:** `WooCommerce account`
- **Connections:**
  - Input ← Create Product Data
  - Output → Check for Product availability
- **Edge cases / failures:**
  - Auth/signature errors (consumer key/secret)
  - If WooCommerce API doesn’t support SKU filtering in the expected way, it may return all products (bad performance)
  - If SKU is empty/null, query may return unexpected results

#### Node: Check for Product availability
- **Type / Role:** IF (`n8n-nodes-base.if`) — decides whether to create or skip.
- **Configuration choices:**
  - Condition checks whether `{{ $json }}` is **empty** (object empty operation).
  - **Interpretation:** Intended logic: “if no product returned → create”; “if found → skip”.
  - **Important caution:** WooCommerce “Get All” often returns an **array of products**. Depending on n8n node output shaping, `$json` may not be “empty” even when no results exist, or it may contain metadata. This is a common source of incorrect branching.
- **Connections:**
  - Input ← Get WooCommerce Products
  - True/First output → **Import Product to WooCommerce**
  - False/Second output → **Loop Over Items** (skip and continue)
- **Edge cases / failures:**
  - Incorrect emptiness check can cause duplicates or prevent creations.
  - If upstream alwaysOutputData produces an item with an empty structure, the condition might behave differently than expected.

---

### 2.6 Create in WooCommerce + Slack Error Handling + Loop Control
**Overview:**  
Creates the WooCommerce product (with images and HTML description). If creation fails, it sends a Slack error notification. Either way, the workflow loops back to continue processing remaining items.

**Nodes involved:**
- Import Product to WooCommerce
- Send Error

#### Node: Import Product to WooCommerce
- **Type / Role:** WooCommerce create product (`n8n-nodes-base.wooCommerce`)
- **Configuration choices:**
  - **Operation:** Create product
  - **Name:** from Create Product Data → `output.title`
  - **SKU:** `output.sku`
  - **Description:** `output.html_body`
  - **Regular price:** `output.price`
  - **Manage stock:** true
  - **Stock quantity:** `0`
  - **Images:** exactly two entries from `output.image_list[0]` and `[1]` with `src`, `name`, `alt`
  - **onError:** `continueErrorOutput` (critical: workflow continues and emits error output)
- **Credentials:** `WooCommerce account`
- **Connections:**
  - Input ← Check for Product availability (create path)
  - Output 0 (success) → **Loop Over Items**
  - Output 1 (error) → **Send Error**
- **Edge cases / failures:**
  - If `image_list` has < 2 items (despite prompt), expressions will fail (index out of range)
  - If WooCommerce rejects `regularPrice` format (currency symbol), create fails
  - Invalid image URLs or blocked hotlinking can fail image import
  - SKU already exists → WooCommerce may reject or create duplicates depending on store behavior/plugins

#### Node: Send Error
- **Type / Role:** Slack message node — notifies team about creation failure.
- **Configuration choices:**
  - Text: `Adding Product Step faces and error - {{ title }} - {{ sku }}`
  - Pulls values from: `$('Create Product Data').first().json.output.title` and `.sku`
  - Channel: `C09KLV9DJSX`
- **Credentials:** `Slack account 2`
- **Connections:**
  - Input ← Import Product to WooCommerce (error output)
  - Output → Loop Over Items (continue processing next Shopify product)
- **Edge cases / failures:**
  - If Create Product Data failed earlier or output missing, Slack message expressions can resolve to empty or throw
  - Slack rate limits if many errors occur rapidly

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Manual Execution | Manual Trigger | Entry point for manual runs | — | Get Shopify Products | ### 🛍️ Step 1: Fetch & Iterate<br>The workflow starts by retrieving all products from the connected Shopify store. It then enters a loop to process each product individually. |
| Get Shopify Products | Shopify | Fetch all products from Shopify | Manual Execution | Loop Over Items | ### 🛍️ Step 1: Fetch & Iterate<br>The workflow starts by retrieving all products from the connected Shopify store. It then enters a loop to process each product individually. |
| Loop Over Items | Split In Batches | Iterate over Shopify products and manage loop-back | Get Shopify Products; Import Product to WooCommerce; Send Error; Check for Product availability | Analyze the Products; Notify Slack Team of Completion | ### 🛍️ Step 1: Fetch & Iterate<br>The workflow starts by retrieving all products from the connected Shopify store. It then enters a loop to process each product individually. |
| Analyze the Products | LangChain Agent | Classify product title → pick BestSite | Loop Over Items | Search for Product Info | ### 🧠 Step 2: AI Classification & Enrichment<br>For each product, an AI agent analyzes the title to extract key product details and determine the best category or "BestSite" reference (e.g., Amazon, Steam). It can optionally trigger a BrowserAct scraper to fetch additional details if needed. |
| Google Gemini | Google Gemini Chat Model | LLM provider for classification agent/parser | — | Analyze the Products; Structured Output | ### 🧠 Step 2: AI Classification & Enrichment<br>For each product, an AI agent analyzes the title to extract key product details and determine the best category or "BestSite" reference (e.g., Amazon, Steam). It can optionally trigger a BrowserAct scraper to fetch additional details if needed. |
| Structured Output | Structured Output Parser | Enforce JSON output for classification | Google Gemini (AI) | Analyze the Products (AI parser link) | ### 🧠 Step 2: AI Classification & Enrichment<br>For each product, an AI agent analyzes the title to extract key product details and determine the best category or "BestSite" reference (e.g., Amazon, Steam). It can optionally trigger a BrowserAct scraper to fetch additional details if needed. |
| Search for Product Info | BrowserAct | Enrich product info via BrowserAct workflow | Analyze the Products | Create Product Data | ### 🧠 Step 2: AI Classification & Enrichment<br>For each product, an AI agent analyzes the title to extract key product details and determine the best category or "BestSite" reference (e.g., Amazon, Steam). It can optionally trigger a BrowserAct scraper to fetch additional details if needed. |
| Create Product Data | LangChain Agent | Generate WooCommerce-ready product JSON | Search for Product Info | Get WooCommerce Products | ### 🧠 Step 2: AI Classification & Enrichment<br>For each product, an AI agent analyzes the title to extract key product details and determine the best category or "BestSite" reference (e.g., Amazon, Steam). It can optionally trigger a BrowserAct scraper to fetch additional details if needed. |
| Google Gemini1 | Google Gemini Chat Model | LLM provider for product data generation | — | Create Product Data; Structured Output1 | ### 🧠 Step 2: AI Classification & Enrichment<br>For each product, an AI agent analyzes the title to extract key product details and determine the best category or "BestSite" reference (e.g., Amazon, Steam). It can optionally trigger a BrowserAct scraper to fetch additional details if needed. |
| Structured Output1 | Structured Output Parser | Enforce JSON output for final product object | Google Gemini1 (AI) | Create Product Data (AI parser link) | ### 🧠 Step 2: AI Classification & Enrichment<br>For each product, an AI agent analyzes the title to extract key product details and determine the best category or "BestSite" reference (e.g., Amazon, Steam). It can optionally trigger a BrowserAct scraper to fetch additional details if needed. |
| Get WooCommerce Products | WooCommerce | Check existence by SKU | Create Product Data | Check for Product availability | ### 🔄 Step 3: Duplicate Check & Sync<br>Before creating a new listing, the system checks WooCommerce to see if a product with the same SKU already exists.<br>* If missing: It creates a new product with the optimized title, description, price, and images.<br>* If found: It skips creation (or updates, depending on configuration) to avoid duplicates. |
| Check for Product availability | IF | Route: create if missing, else skip | Get WooCommerce Products | Import Product to WooCommerce; Loop Over Items | ### 🔄 Step 3: Duplicate Check & Sync<br>Before creating a new listing, the system checks WooCommerce to see if a product with the same SKU already exists.<br>* If missing: It creates a new product with the optimized title, description, price, and images.<br>* If found: It skips creation (or updates, depending on configuration) to avoid duplicates. |
| Import Product to WooCommerce | WooCommerce | Create product in WooCommerce | Check for Product availability | Loop Over Items; Send Error | ### 🔄 Step 3: Duplicate Check & Sync<br>Before creating a new listing, the system checks WooCommerce to see if a product with the same SKU already exists.<br>* If missing: It creates a new product with the optimized title, description, price, and images.<br>* If found: It skips creation (or updates, depending on configuration) to avoid duplicates. |
| Send Error | Slack | Post error to Slack and continue loop | Import Product to WooCommerce (error output) | Loop Over Items | ### 🔄 Step 3: Duplicate Check & Sync<br>Before creating a new listing, the system checks WooCommerce to see if a product with the same SKU already exists.<br>* If missing: It creates a new product with the optimized title, description, price, and images.<br>* If found: It skips creation (or updates, depending on configuration) to avoid duplicates. |
| Notify Slack Team of Completion | Slack | Post completion message | Loop Over Items (completion path) | — |  |
| Documentation | Sticky Note | Comment/documentation | — | — | ## ⚡ Workflow Overview & Setup<br>**Summary:** This automation synchronizes new products from a Shopify store to a WooCommerce store. It intelligently checks for existing products to prevent duplicates, uses AI to categorize and refine product data, and notifies the team via Slack upon completion or error.<br><br>### Requirements<br>* **Credentials:** Shopify, WooCommerce, BrowserAct, Google Gemini (PaLM), Slack.<br>* **Mandatory:** BrowserAct API (Template: **Shopify to WooCommerce Multi-Store Sync** )<br><br>### How to Use<br>1. **Credentials:** Connect your Shopify Admin, WooCommerce API, Slack, and AI accounts in n8n.<br>2. **BrowserAct Template:** Ensure you have the **Shopify to WooCommerce Multi-Store Sync** template saved if you plan to use the enrichment features.<br>3. **Trigger:** The workflow can be triggered manually or scheduled (needs a schedule node added) to fetch all products from Shopify.<br><br>### Need Help?<br>[How to Find Your BrowserAct API Key & Workflow ID](https://docs.browseract.com)<br>[How to Connect n8n to BrowserAct](https://docs.browseract.com)<br>[How to Use & Customize BrowserAct Templates](https://docs.browseract.com) |
| Step 1 Explanation | Sticky Note | Comment/documentation | — | — | ### 🛍️ Step 1: Fetch & Iterate<br>The workflow starts by retrieving all products from the connected Shopify store. It then enters a loop to process each product individually. |
| Step 2 Explanation | Sticky Note | Comment/documentation | — | — | ### 🧠 Step 2: AI Classification & Enrichment<br>For each product, an AI agent analyzes the title to extract key product details and determine the best category or "BestSite" reference (e.g., Amazon, Steam). It can optionally trigger a BrowserAct scraper to fetch additional details if needed. |
| Step 3 Explanation | Sticky Note | Comment/documentation | — | — | ### 🔄 Step 3: Duplicate Check & Sync<br>Before creating a new listing, the system checks WooCommerce to see if a product with the same SKU already exists.<br>* If missing: It creates a new product with the optimized title, description, price, and images.<br>* If found: It skips creation (or updates, depending on configuration) to avoid duplicates. |
| Sticky Note | Sticky Note | Comment/link | — | — | @[youtube](Ad-Wy9bNVGw) |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name it: **“Sync Shopify products to WooCommerce with Gemini, BrowserAct & Slack”**
- Ensure workflow setting **Execution Order** is default (v1 is fine).

2) **Add Trigger**
- Add node: **Manual Trigger**  
  - Name: `Manual Execution`

3) **Fetch Shopify products**
- Add node: **Shopify**
  - Name: `Get Shopify Products`
  - Resource: **Product**
  - Operation: **Get All**
  - Return All: **true**
  - Authentication: **Access Token**
- **Credentials:** Create/attach a Shopify access token credential with Admin API access.
- Connect: `Manual Execution` → `Get Shopify Products`

4) **Add batching loop**
- Add node: **Split In Batches**
  - Name: `Loop Over Items`
  - Use default settings (or set a batch size you prefer).
- Connect: `Get Shopify Products` → `Loop Over Items`

5) **Completion notification**
- Add node: **Slack**
  - Name: `Notify Slack Team of Completion`
  - Operation: **Post Message**
  - Select: **Channel**
  - Channel: pick your channel (in JSON: `all-browseract-workflow-test`)
  - Text: `Product sync from Shopify to WooCommerce is done.`
- Connect: `Loop Over Items` (done/no items output) → `Notify Slack Team of Completion`

6) **Gemini model for classification**
- Add node: **Google Gemini Chat Model** (LangChain)
  - Name: `Google Gemini`
  - Model: `models/gemini-2.5-pro`
- **Credentials:** Google Gemini (PaLM) API credential in n8n.

7) **Structured output parser (classification)**
- Add node: **Structured Output Parser** (LangChain)
  - Name: `Structured Output`
  - Auto-fix: **enabled**
  - Schema example (enter the example used in the node):
    - Keys: `Type`, `Products`, `BestSite`

8) **Classification agent**
- Add node: **AI Agent** (LangChain)
  - Name: `Analyze the Products`
  - Text: `{{ $json.title }}`
  - Prompt type: “Define” (or equivalent in your n8n version)
  - System message: paste the strict classification rules (as in workflow)
  - Enable “Use Output Parser” and select **Structured Output**
  - Select **Google Gemini** as the model
- Connect: `Loop Over Items` (per-item output) → `Analyze the Products`
- Ensure AI connections:
  - `Google Gemini` → (AI language model) `Analyze the Products`
  - `Google Gemini` → (AI language model) `Structured Output`
  - `Structured Output` → (AI output parser) `Analyze the Products`

9) **BrowserAct enrichment**
- Add node: **BrowserAct**
  - Name: `Search for Product Info`
  - Type: **WORKFLOW**
  - Workflow ID: `70444920944876283`
  - Timeout: `7200`
  - Map inputs:
    - `input-Product` = `{{ $json.output.Products }}`
    - `input-Target_Site` = `{{ $json.output.BestSite }}`
- **Credentials:** BrowserAct API credential (API key).
- Connect: `Analyze the Products` → `Search for Product Info`

10) **Gemini model for product data generation**
- Add node: **Google Gemini Chat Model**
  - Name: `Google Gemini1`
  - Model: `models/gemini-2.5-pro`
- Reuse same Gemini credential or another, as desired.

11) **Structured output parser (product JSON)**
- Add node: **Structured Output Parser**
  - Name: `Structured Output1`
  - Auto-fix: **enabled**
  - Schema example: must include `title`, `sku`, `price`, `html_body`, `image_list[] {url,name,alt}`.

12) **Product data creation agent**
- Add node: **AI Agent**
  - Name: `Create Product Data`
  - Text: `{{ $json.output.string }}`
  - System message: paste the “E-Commerce Data Architect and Senior Copywriter” constraints (as in workflow), including “always add two images”.
  - Enable output parser and set to **Structured Output1**
  - Select **Google Gemini1** as model
- Connect: `Search for Product Info` → `Create Product Data`
- Ensure AI connections:
  - `Google Gemini1` → (AI language model) `Create Product Data`
  - `Google Gemini1` → (AI language model) `Structured Output1`
  - `Structured Output1` → (AI output parser) `Create Product Data`

13) **WooCommerce lookup by SKU**
- Add node: **WooCommerce**
  - Name: `Get WooCommerce Products`
  - Operation: **Get All** (Products)
  - Return all: **true**
  - Add an SKU filter/option (depending on node UI):
    - `sku` = `{{ $json.output.sku }}`
  - Enable **Always Output Data** (so the IF node can decide even when no match).
- **Credentials:** WooCommerce REST API credentials (consumer key/secret) for your store.
- Connect: `Create Product Data` → `Get WooCommerce Products`

14) **IF node to decide create vs skip**
- Add node: **IF**
  - Name: `Check for Product availability`
  - Condition: check whether the incoming data is empty (replicate the current logic).
  - Connect:
    - True/create path → `Import Product to WooCommerce`
    - False/skip path → `Loop Over Items` (to continue)

15) **Create WooCommerce product**
- Add node: **WooCommerce**
  - Name: `Import Product to WooCommerce`
  - Resource: **Product**
  - Operation: **Create**
  - Name: `{{ $('Create Product Data').first().json.output.title }}`
  - SKU: `{{ $('Create Product Data').first().json.output.sku }}`
  - Description: `{{ $('Create Product Data').first().json.output.html_body }}`
  - Regular price: `{{ $('Create Product Data').first().json.output.price }}`
  - Manage stock: **true**
  - Stock quantity: `0`
  - Images: add two image entries:
    - Image 1: src/name/alt from `image_list[0]`
    - Image 2: src/name/alt from `image_list[1]`
  - Set node error handling to **Continue on Fail** / `continueErrorOutput` equivalent.
- Connect success output → `Loop Over Items`

16) **Slack error notification**
- Add node: **Slack**
  - Name: `Send Error`
  - Post message to same channel
  - Text: `Adding Product Step faces and error - {{ $('Create Product Data').first().json.output.title }} - {{ $('Create Product Data').first().json.output.sku }}`
- Connect:
  - `Import Product to WooCommerce` error output → `Send Error`
  - `Send Error` → `Loop Over Items`

17) **(Optional) Add scheduling**
- Add **Schedule Trigger** instead of manual trigger (or in addition with multiple entry points), if you want periodic sync.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| BrowserAct documentation links for API key, n8n connection, and templates | https://docs.browseract.com |
| BrowserAct template requirement: **Shopify to WooCommerce Multi-Store Sync** | Mentioned in workflow notes (must exist in BrowserAct if using enrichment) |
| YouTube reference | `@[youtube](Ad-Wy9bNVGw)` |
| Completion + error notifications are sent to Slack channel `all-browseract-workflow-test` | Adjust channel and permissions as needed |


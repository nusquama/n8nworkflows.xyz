Generate Shopify collection blog posts with Perplexity, Gemini and Google Sheets

https://n8nworkflows.xyz/workflows/generate-shopify-collection-blog-posts-with-perplexity--gemini-and-google-sheets-12694


# Generate Shopify collection blog posts with Perplexity, Gemini and Google Sheets

## 1. Workflow Overview

**Workflow title:** Generate Shopify collection blog posts with Perplexity, Gemini and Google Sheets

**Purpose:**  
This workflow builds an end-to-end automation that (1) captures Shopify Collection metadata (new and existing), (2) tracks processing state in Google Sheets, (3) uses Perplexity for structured research and Gemini (LangChain agent) to generate long-form blog content, (4) generates a relevant image with Gemini Image, uploads it to Shopify Files via staged uploads, and (5) publishes a Shopify Blog Article and writes back the final article URL to Google Sheets.

**Primary use cases:**
- Automatically turn new Shopify collections into SEO-focused buying-guide style blog articles.
- Backfill all existing Shopify collections into a tracking sheet and process them in batches.
- Maintain a content production pipeline with statuses (pending → generated → image updated → sent for approval → posted).

### 1.1 Collection Intake & Tracking (New + Backfill)
- New collection created (Shopify Trigger) → store in Google Sheet as “just added”.
- Manual run fetches collections from Shopify GraphQL → store in Google Sheet as “old collections”.

### 1.2 Blog Content Generation (Research + Article Draft)
- On schedule, pick “pending” rows → Perplexity research JSON → Gemini agent generates article JSON → Code builds HTML → update sheet (status “generated”).

### 1.3 Image Generation + Shopify File Upload
- On schedule, pick “generated” rows missing image_url → Gemini image → Shopify staged upload → create File → poll for READY → update sheet (status “image updated”, image_url saved).

### 1.4 Publish to Shopify Blog + Final Tracking
- Pick “image updated” rows → replace placeholder image URL in HTML → update sheet to “sent for approval” (holds final HTML).
- Pick “sent for approval” rows → create Shopify article in a specific blog → query article details via GraphQL → compute article URL → update sheet to “posted”.

### 1.5 One-time Blog Creation (Optional)
- Scheduled block to create a Shopify Blog (intended “one time only”).

---

## 2. Block-by-Block Analysis

### Block 2.1 — New Collection Intake (Realtime)
**Overview:** When a new Shopify collection is created, the workflow normalizes its fields and appends it to Google Sheets as “just added” with status “pending”.

**Nodes involved:**
- Shopify Trigger1
- data allocation
- collection data storing1
- 2 Sec delay
- get old collection

#### Node: Shopify Trigger1
- **Type / role:** `Shopify Trigger` — webhook entrypoint for Shopify events.
- **Configuration:**  
  - Topic: `collections/create`  
  - Auth: Shopify Admin Access Token (predefined credential).
- **Input/Output:** Entry node → outputs Shopify event payload.
- **Key data usage:** Downstream assumes collection data under `$json.node.*`.
- **Failure/edge cases:**  
  - Webhook delivery failures, Shopify permission scope issues.  
  - Payload shape mismatch (if Shopify trigger outputs `collection` instead of `node` in some versions/stores).

#### Node: data allocation
- **Type / role:** `Set` — maps Shopify payload to a simplified schema.
- **Configuration choices:** Creates fields: `id, title, handle, description, updatedAt` from:
  - `={{ $json.node.id }}`
  - `={{ $json.node.title }}` etc.
- **Input/Output:** Shopify Trigger1 → data allocation → Google Sheets append.
- **Edge cases:** If any field is missing/null, downstream sheet columns may receive empty strings.

#### Node: collection data storing1
- **Type / role:** `Google Sheets` — append a row for the newly created collection.
- **Configuration choices:**  
  - Operation: **Append**  
  - Matching columns configured as `id`, but append does not deduplicate; it only uses schema mapping.
  - Writes:
    - `type = "just added"`
    - `status = "pending"`
    - `collection_url = "=www.astools.nl/collections/{{ $json.handle }}"` (note: stored as a sheet formula-like string; missing protocol `https://`).
- **Credentials:** Google Sheets OAuth2 (`n8n.verified@gmail.com`).
- **Failure/edge cases:**  
  - Duplicate IDs can be appended repeatedly (no pre-check).  
  - If the sheet expects a real URL, the missing protocol may cause non-clickable links.  
  - Google API quota/auth expiration.

#### Node: 2 Sec delay
- **Type / role:** `Wait` — pauses 2 seconds before reading the sheet.
- **Purpose:** Likely to allow Google Sheets append to settle before querying.
- **Edge cases:** Not a guarantee; still possible to read before consistency is achieved.

#### Node: get old collection
- **Type / role:** `Google Sheets` — retrieves first matching row.
- **Configuration:** Filter:
  - `type == "just added"`
  - `status == "pending"`
  - Option: `returnFirstMatch = true`
- **Output usage:** Feeds into scheduled content generation path via `data allocation8`.
- **Edge cases:**  
  - If multiple pending “just added” rows exist, only the first match is processed per run of this path.

---

### Block 2.2 — Backfill Existing Collections (Manual)
**Overview:** Manual trigger fetches Shopify collections via GraphQL, splits them into items, maps fields, and appends them to the sheet as “old collections”.

**Nodes involved:**
- START
- collection API call
- devide items
- batch size 10
- data allocation1
- collection data storing

#### Node: START
- **Type / role:** `Manual Trigger` — manual entrypoint for backfill.
- **Edge cases:** None (human-run).

#### Node: collection API call
- **Type / role:** `HTTP Request` — Shopify GraphQL query to list collections.
- **Configuration choices:**
  - POST `https://{store-name}.myshopify.com/admin/api/2025-10/graphql.json`
  - Body includes query `GetCollections(first: 2, after: null)`
  - Auth: predefined Shopify Access Token credential.
- **Important limitation:** Pagination (`hasNextPage/endCursor`) is returned but **not looped**. Only `first = 2` collections are retrieved per manual run.
- **Edge cases:**  
  - Wrong Admin API version or store name placeholder not replaced.  
  - GraphQL errors not handled (no IF on userErrors).  
  - Shopify rate limits.

#### Node: devide items
- **Type / role:** `Split Out` — splits array into individual items.
- **Configuration:** `fieldToSplitOut = "data.collections.edges"`
- **Output:** Each item becomes one edge object with `.node`.
- **Edge cases:** If Shopify returns errors or different structure, split fails.

#### Node: batch size 10
- **Type / role:** `Split In Batches` — processes items in batches of 10.
- **Configuration:** `batchSize = 10`
- **Connections detail:** Uses output index 1 to send items onward to `data allocation1`. (This is unusual; ensure the “Batches” node is connected correctly in the UI.)
- **Edge cases:** If miswired, it may not emit items as expected.

#### Node: data allocation1
- **Type / role:** `Set` — same mapping as “data allocation” from `$json.node.*`.
- **Edge cases:** Same as above.

#### Node: collection data storing
- **Type / role:** `Google Sheets` — append rows for old collections.
- **Configuration choices:**  
  - Operation: Append  
  - Writes:
    - `type = "old collections"`
    - `status = "pending"`
    - `collection_url = "=www.astools.nl/collections/{{ $json.handle }}"`
- **Edge cases:** Duplicates possible; no “upsert” logic.

---

### Block 2.3 — Scheduled Article Draft Generation (Perplexity → Gemini Agent → HTML → Sheet Update)
**Overview:** On schedule, the workflow selects a “pending” collection (old collections), performs structured market/SEO research with Perplexity, generates an article JSON with a Gemini-based agent, converts it to HTML, and updates the sheet to “generated” with tags/title/image prompt.

**Nodes involved:**
- time 3
- get old collection1
- data allocation8
- Limiter
- set batch size
- delay
- web search2
- content generator2
- Memory2
- Gemini2
- Structured2
- Output structure
- Gemini5
- HTML structuring3
- update content3

#### Node: time 3
- **Type / role:** `Schedule Trigger`
- **Configuration:** `rule.interval` is present but empty (`[{}]`). In n8n this often means misconfigured schedule; verify in UI.
- **Edge cases:** Trigger may not run at all until interval is set (e.g., every X minutes).

#### Node: get old collection1
- **Type / role:** `Google Sheets` — fetch first match pending old collection.
- **Filters:** `type = "old collections"`, `status = "pending"`, `returnFirstMatch = true`.
- **Edge cases:** Only processes one row per trigger execution.

#### Node: data allocation8
- **Type / role:** `Set` — normalizes row fields for AI steps.
- **Fields:** `id,title,handle,description,updatedAt,type,status,collection_url` from sheet row.

#### Node: Limiter
- **Type / role:** `Limit` — restrict items passing through.
- **Configuration:** default (likely 1). Acts as safety to prevent large runs.

#### Node: set batch size
- **Type / role:** `Split In Batches`
- **Role:** Controls sequential AI processing; connected to both:
  - `delay` (image pipeline kickoff later)
  - `web search2` (research pipeline)
- **Edge cases:** With only one input item (because returnFirstMatch + Limit), it mainly acts as a control node.

#### Node: delay
- **Type / role:** `Wait` — 2 seconds.
- **Connection:** `set batch size` → `delay` → `get img prompt` (image pipeline reads generated rows later).  
  This creates a coupling between text-generation and image-generation scheduling; ensure it does not cause premature image runs before `generated` is written.

#### Node: web search2
- **Type / role:** `Perplexity` — structured research agent.
- **Configuration choices:**
  - Sends a single message prompt that:
    - Uses `{{ $json.title }}`, `{{ $json.description }}`, `{{ $json.collection_url }}`
    - Requires output as strict JSON with schema (collectionSummary, seoResearch, etc.)
  - `simplify = true`
- **Output:** `message` (stringified JSON or parsed depending on node behavior).
- **Edge cases:**  
  - If Perplexity returns non-JSON or extra text, downstream may fail.  
  - API quota/auth issues.

#### Node: content generator2
- **Type / role:** `LangChain Agent` — generates the blog article JSON.
- **Configuration choices:**
  - **Text input:** Includes:
    - `product_title` and `product_description` from `$('set batch size').item.json.*` (explicit cross-node reference)
    - `websearch_data : {{ $json.message }}`
  - **System message:** Enforces strict JSON output, long-form schema, conversion-focused rules.
  - `hasOutputParser = true`
- **AI dependencies (connections):**
  - Language model input from **Gemini2**
  - Memory from **Memory2**
  - Output parser from **Output structure** (which itself uses Structured2 + Gemini5)
- **Edge cases:**  
  - The agent prompt says “use provided research JSON as ONLY factual source”, but it also receives product title/description; ambiguity can cause model to mix sources.  
  - If Perplexity output is a string containing JSON, agent may treat it as plain text and deviate.

#### Node: Gemini2
- **Type / role:** `Google Gemini Chat Model` used as LLM for the agent.
- **Model:** `models/gemini-3-flash-preview`
- **Edge cases:** Model availability changes; preview models may be deprecated.

#### Node: Memory2
- **Type / role:** `Memory Buffer Window`
- **Configuration:** `sessionKey = {{ $now.minute }}` custom key.
- **Risk:** All runs within the same minute share memory; cross-contamination possible if multiple items run in the same minute.

#### Node: Structured2
- **Type / role:** `Structured Output Parser` (manual schema)
- **Schema:** Matches the required article JSON fields.
- **Role:** Validates/structures agent outputs before passing to autofixing.

#### Node: Gemini5
- **Type / role:** `Google Gemini Chat Model` used by autofixing parser.
- **Model:** Not explicitly set in node parameters (uses default in credential/node UI).
- **Edge cases:** Default model changes can affect parsing reliability.

#### Node: Output structure
- **Type / role:** `Autofixing Output Parser`
- **Role:** Attempts to fix JSON to fit schema using Gemini5 + Structured2.
- **Edge cases:** If output is severely malformed, it may still fail or “fix” incorrectly.

#### Node: HTML structuring3
- **Type / role:** `Code` — converts the structured article JSON into Shopify-ready HTML.
- **Key logic:**
  - Reads: `const data = $input.first().json.output;` (expects agent output nested under `output`)
  - Builds HTML sections: introduction, problem, solution, tables, lists, checklist, CTA
  - Injects an `<img src="IMAGE_REPLACE">` placeholder to be replaced later.
  - Outputs JSON:
    - `body_html`
    - `article_title`
    - `seo_tags` (array joined with commas)
    - `image_prompt = data.image_generation_prompt`
- **Edge cases:**  
  - If the agent output is not under `.output`, this node fails.  
  - `renderFeaturesTable` expects `features_and_benefits` items shaped `{name,value}` (prompt says that), ok.  
  - Some HTML uses `<table border=\"1\">` etc.; Shopify may sanitize some tags depending on theme/settings.

#### Node: update content3
- **Type / role:** `Google Sheets` — updates the row to store generated content.
- **Operation:** Update, matching by `id`.
- **Writes:**
  - `status = "generated"`
  - `ai_tags = {{ $json.seo_tags }}`
  - `ai_title = {{ $json.article_title }}`
  - `image_prompt = {{ $json.image_prompt }}`
  - `ai_description = {{ $json.body_html }}` (note: stored as HTML)
- **Important bug:** It references `{{ $('set batch size').item.json.id }}` for `id`, while updating sheet. If the batch context differs or multiple items, it can mismatch.
- **Edge cases:** Large HTML may exceed Google Sheets cell limits (50k chars limit for a cell).

---

### Block 2.4 — Scheduled Blog Creation (One-time)
**Overview:** Creates a Shopify Blog titled “Collections Blog”. Intended to be run once.

**Nodes involved:**
- time 6
- data allocation14
- article creation7

#### Node: time 6
- **Type / role:** `Schedule Trigger`
- **Configuration:** interval array empty (`[{}]`) — verify schedule.
- **Edge cases:** Might run repeatedly unintentionally if schedule is later configured.

#### Node: data allocation14
- **Type / role:** `Set`
- **Writes constants:**
  - `title = "Collections Blog"`
  - `handle = "collections-blog"`

#### Node: article creation7
- **Type / role:** `HTTP Request` — Shopify GraphQL `blogCreate`.
- **Endpoint:** `https://{store-name}.myshopify.com/admin/api/2026-01/graphql.json`
- **Mutation:** Creates blog with templateSuffix `standard`, commentPolicy `MODERATED`.
- **Edge cases:**  
  - If blog already exists with same handle, Shopify returns userErrors; not handled.  
  - API version mismatch.

---

### Block 2.5 — Scheduled Image Generation + Upload to Shopify Files
**Overview:** For generated articles that do not yet have an image_url, the workflow generates an image using Gemini Image, uploads to Shopify via staged upload, creates a File, polls until READY, then writes the final image URL back to Google Sheets.

**Nodes involved:**
- delay
- get img prompt
- set batch size1
- delay2
- create space
- image creation
- update space
- get temp url
- delay1
- get img url
- If verified
- Update row in sheet

#### Node: get img prompt
- **Type / role:** `Google Sheets` — selects items ready for image generation.
- **Filters:**
  - `status == "generated"`
  - `image_url` filter present with only `lookupColumn` (no value). This typically means “is empty” in UI, but confirm; otherwise it may behave unexpectedly.
- **executeOnce:** true (node-level). This means in production it may only execute once per workflow activation depending on n8n behavior; verify, as it can block ongoing runs.

#### Node: set batch size1
- **Type / role:** `Split In Batches`
- **Outputs:**
  - Output 0 → delay2 → later publishing pipeline reads “image updated”
  - Output 1 → create space → image generation pipeline
- **Edge cases:** Ensure correct wiring; otherwise it might skip image creation.

#### Node: create space
- **Type / role:** `HTTP Request` — Shopify GraphQL `stagedUploadsCreate`.
- **Purpose:** Get signed upload parameters for Google storage.
- **Variables:** filename uses `{store-name}{{ $json.id }}`; `{store-name}` must be replaced.
- **Edge cases:**  
  - Wrong resource type (`PRODUCT_IMAGE`) but used for blog image; may still work but not ideal.  
  - userErrors not handled.

#### Node: image creation
- **Type / role:** `Google Gemini (Image)` via LangChain node.
- **Model:** `models/gemini-2.5-flash-image`
- **Prompt:** `={{ $('set batch size1').item.json.image_prompt }}`
- **Output:** binary image data field named `data` (assumed by multipart upload).
- **Edge cases:** Prompt may be empty; model may return safety blocks; binary field name mismatch breaks upload.

#### Node: update space
- **Type / role:** `HTTP Request` — multipart upload to `https://shopify-staged-uploads.storage.googleapis.com`
- **Configuration:** Uses form fields from `create space` response; uploads binary `file` from input field `data`.
- **Edge cases:**  
  - If any parameter index changes in Shopify response, expressions like `parameters[7]` break.  
  - Upload returns XML; downstream parsing expects `<Location>...</Location>`.

#### Node: get temp url
- **Type / role:** `HTTP Request` — Shopify GraphQL `fileCreate`.
- **Key expression:** Extracts uploaded file location from XML via regex:
  - `{{ $json.data.match(/<Location>(.*?)<\/Location>/)[1] }}`
- **Edge cases:** If upload response format changes or no `<Location>`, this throws.

#### Node: delay1
- **Type / role:** `Wait` — no explicit amount set (defaults to 1 minute in some versions, or 0; must be verified in UI).
- **Role:** Gives Shopify time to process File before querying preview URL.

#### Node: get img url
- **Type / role:** `HTTP Request` — Shopify GraphQL query to fetch file status and preview URL.
- **Query:** `node(id: "...") { ... on File { fileStatus preview { image { url }}}}`
- **Edge cases:** preview may be null until processing complete.

#### Node: If verified
- **Type / role:** `IF` — checks `fileStatus == READY`.
- **True path:** Update row in sheet with image URL.
- **False path:** Goes to delay1 (polling loop).
- **Edge cases:** Infinite loop risk if file never becomes READY; no max retries.

#### Node: Update row in sheet
- **Type / role:** `Google Sheets` — update image_url and status.
- **Writes:**
  - `status = "image updated"`
  - `image_url = {{ $json.data.node.preview.image.url }}`
  - `id = {{ $('set batch size1').item.json.id }}`
- **Edge cases:** If preview is missing, writes blank and still marks updated (depending on data).

---

### Block 2.6 — Insert Image into HTML and Mark “Sent for approval”
**Overview:** Takes rows where image is updated, replaces the `IMAGE_REPLACE` placeholder inside the stored HTML, and updates the sheet to “sent for approval”.

**Nodes involved:**
- delay2
- get content (img updated)
- set batch size2
- replace img url
- update content4

#### Node: delay2
- **Type / role:** `Wait` — 2 seconds.
- **Role:** Delays before fetching “image updated” rows.

#### Node: get content (img updated)
- **Type / role:** `Google Sheets`
- **Filter:** `status == "image updated"`
- **executeOnce:** true (risk similar to “get img prompt”).
- **Edge cases:** If multiple rows match, it returns multiple items unless limited elsewhere.

#### Node: set batch size2
- **Type / role:** `Split In Batches`
- **Outputs:**
  - Output 0 → get old collection6 (publishing)
  - Output 1 → replace img url (HTML update)
- **Edge cases:** Wiring must be verified; otherwise publishing might start before HTML is updated.

#### Node: replace img url
- **Type / role:** `Code`
- **Logic:** Replace all occurrences of `IMAGE_REPLACE` with `image_url`:
  - `ai_description.replace(/IMAGE_REPLACE/g, image_url)`
- **Inputs expected:** `ai_description` and `image_url` from sheet row.
- **Output:** `{ body_html: updated_html }`
- **Edge cases:** If HTML does not contain placeholder, no change. If image_url is empty, placeholder replaced with empty string.

#### Node: update content4
- **Type / role:** `Google Sheets` — update with final HTML (with image).
- **Writes:**
  - `status = "sent for approval"`
  - `ai_description = {{ $json.body_html }}`
  - `id = {{ $('set batch size2').item.json.id }}`
- **Edge cases:** Same cell size limit risk.

---

### Block 2.7 — Publish Shopify Blog Article and Store Final URL
**Overview:** Creates a Shopify article from the prepared HTML, then queries Shopify GraphQL to compute the final public URL and updates the sheet to “posted”.

**Nodes involved:**
- get old collection6
- data allocation15
- article creation8
- data allocation16
- article creation9
- Edit Fields
- get just added1

#### Node: get old collection6
- **Type / role:** `Google Sheets`
- **Filters:** `type == "old collections"`, `status == "sent for approval"`, `returnFirstMatch = true`
- **Edge cases:** Only one row published per run.

#### Node: data allocation15
- **Type / role:** `Set` — maps sheet fields to Shopify REST article create payload.
- **Writes:**
  - `title = {{ $json.ai_title }}`
  - `author = "AsTools"`
  - `tags = {{ $json.ai_tags }}`
  - `body_html = {{ $json.ai_description }}`
  - `published_at = {{ $now.toFormat('dd/MM/yyyy') }}` (note: Shopify expects ISO 8601; this may be rejected or interpreted incorrectly)
  - `blog_id = "{blog-id}"` placeholder must be replaced with actual blog numeric ID for REST endpoint.
- **Edge cases:** Date format likely invalid for Shopify REST `published_at`.

#### Node: article creation8
- **Type / role:** `HTTP Request` — Shopify REST create article.
- **Endpoint:** `https://{store-name}.myshopify.com/admin/api/2025-07/blogs/{{ $json.blog_id }}/articles.json`
- **Body:** `{ article: { title, author, tags, body_html, published_at } }`
- **Edge cases:**  
  - `{blog-id}` not replaced → 404.  
  - Admin API version mismatch.  
  - HTML rejected/sanitized.  
  - Tags format: Shopify accepts comma-separated string; here it’s passed as-is from sheet.

#### Node: data allocation16
- **Type / role:** `Set` — extracts GraphQL admin ID from REST response.
- **Field:** `article_id = {{ $json.article.admin_graphql_api_id }}`
- **Edge cases:** If REST response differs, field missing.

#### Node: article creation9
- **Type / role:** `HTTP Request` — Shopify GraphQL query ArticleShow.
- **Endpoint:** hardcoded domain `https://nl-astools.myshopify.com/admin/api/2026-01/graphql.json` (note: not placeholder like others).
- **Returns:** primary domain URL, article handle, blog handle, etc.
- **Edge cases:** If store domain differs, this breaks. API version mismatch.

#### Node: Edit Fields
- **Type / role:** `Set` — composes the public article URL:
  - `={{ $json.data.shop.primaryDomain.url }}/blogs/{{ $json.data.article.blog.handle }}/{{ $json.data.article.handle }}`
- **Edge cases:** If article not published yet or blog handle missing.

#### Node: get just added1
- **Type / role:** `Google Sheets` — update final status and URL.
- **Operation:** Update matching `id`.
- **Writes:**
  - `status = "posted"`
  - `article_url = {{ $json.url }}`
  - `id = {{ $('get old collection6').item.json.id }}`
- **Edge cases:** Mismatch if multiple parallel items; this depends on “first match” behavior.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Shopify Trigger1 | shopifyTrigger | Realtime trigger on collection creation | — | data allocation | # 1. Collect new collection data and old collection data to sheet also create a custom blog for collection if required. / It will automatic start working to collecting **collection listing details** from the shopify store and maintain the google sheet. |
| data allocation | set | Normalize new collection payload | Shopify Trigger1 | collection data storing1 | # 1. Collect new collection data and old collection data to sheet also create a custom blog for collection if required. / ## When new data added to Store |
| collection data storing1 | googleSheets | Append “just added” collection row | data allocation | 2 Sec delay | # 1… / ## When new data added to Store |
| 2 Sec delay | wait | Small delay before reading sheet | collection data storing1 | get old collection | # 1… / ## When new data added to Store |
| get old collection | googleSheets | Fetch first pending “just added” row | 2 Sec delay | data allocation8 | # 1… / ## When new data added to Store |
| START | manualTrigger | Manual entrypoint for backfill | — | collection API call | # 1… / ## All previous collection data available in Shopify Store |
| collection API call | httpRequest | Shopify GraphQL list collections (limited) | START | devide items | # 1… / ## All previous collection data available in Shopify Store |
| devide items | splitOut | Split collections edges into items | collection API call | batch size 10 | # 1… / ## All previous collection data available in Shopify Store |
| batch size 10 | splitInBatches | Batch processing for backfill | devide items | data allocation1 | # 1… / ## All previous collection data available in Shopify Store |
| data allocation1 | set | Normalize old collection edge node | batch size 10 | collection data storing | # 1… / ## All previous collection data available in Shopify Store |
| collection data storing | googleSheets | Append “old collections” rows | data allocation1 | batch size 10 | # 1… / ## All previous collection data available in Shopify Store |
| time 6 | scheduleTrigger | Trigger blog creation (intended one-time) | — | data allocation14 | ## Blog creation (One time only) |
| data allocation14 | set | Set blog title/handle constants | time 6 | article creation7 | ## Blog creation (One time only) |
| article creation7 | httpRequest | Shopify GraphQL blogCreate | data allocation14 | — | ## Blog creation (One time only) |
| time 3 | scheduleTrigger | Trigger draft generation | — | get old collection1 | # 2. Create blog article using collection data |
| get old collection1 | googleSheets | Fetch first pending old collection | time 3 | data allocation8 | # 2… |
| data allocation8 | set | Normalize sheet row for AI processing | get old collection1, get old collection | Limiter | # 2… |
| Limiter | limit | Limit number of processed items | data allocation8 | set batch size | # 2… |
| set batch size | splitInBatches | Controls sequential AI generation | Limiter | delay; web search2 | # 2… |
| web search2 | perplexity | Produce structured research JSON | set batch size | content generator2 | # 2… |
| Gemini2 | lmChatGoogleGemini | LLM for the LangChain agent | — | content generator2 (ai_languageModel) | # 2… |
| Memory2 | memoryBufferWindow | Short-term memory for agent | — | content generator2 (ai_memory) | # 2… |
| Structured2 | outputParserStructured | Defines expected article JSON schema | — | Output structure (ai_outputParser) | # 2… |
| Gemini5 | lmChatGoogleGemini | LLM used by autofixing parser | — | Output structure (ai_languageModel) | # 2… |
| Output structure | outputParserAutofixing | Attempts to fix/validate agent output JSON | Gemini5, Structured2 | content generator2 (ai_outputParser) | # 2… |
| content generator2 | langchain.agent | Generates article JSON from research | web search2 | HTML structuring3 | # 2… |
| HTML structuring3 | code | Convert article JSON → HTML with image placeholder | content generator2 | update content3 | # 2… |
| update content3 | googleSheets | Update sheet to “generated” + store HTML/prompt | HTML structuring3 | set batch size | # 2… |
| delay | wait | Delay before starting image generation fetch | set batch size | get img prompt | # 3. Generate a relevant image for the blog article. |
| get img prompt | googleSheets | Fetch “generated” rows missing image_url | delay | set batch size1 | # 3… |
| set batch size1 | splitInBatches | Controls sequential image generation | get img prompt | delay2; create space | # 3… |
| create space | httpRequest | Shopify stagedUploadsCreate | set batch size1 | image creation | # 3… |
| image creation | googleGemini (image) | Generate image binary from prompt | create space | update space | # 3… |
| update space | httpRequest | Upload image to staged Google URL | image creation | get temp url | # 3… |
| get temp url | httpRequest | Shopify fileCreate using upload Location | update space | delay1 | # 3… |
| delay1 | wait | Wait before querying file preview URL | get temp url, If verified (false) | get img url | # 3… |
| get img url | httpRequest | Query File node preview URL + status | delay1 | If verified | # 3… |
| If verified | if | Poll until fileStatus == READY | get img url | Update row in sheet; delay1 | # 3… |
| Update row in sheet | googleSheets | Store image_url and set status “image updated” | If verified (true) | set batch size1 | # 3… |
| delay2 | wait | Delay before fetching image-updated HTML | set batch size1 | get content (img updated) | # 4. Update image url to article body HTML and post to Shopify |
| get content (img updated) | googleSheets | Fetch rows with status “image updated” | delay2 | set batch size2 | # 4… |
| set batch size2 | splitInBatches | Controls sequential publish prep/publish | get content (img updated) | get old collection6; replace img url | # 4… |
| replace img url | code | Replace IMAGE_REPLACE with real image URL | set batch size2 | update content4 | # 4… |
| update content4 | googleSheets | Save final HTML and mark “sent for approval” | replace img url | set batch size2 | # 4… |
| get old collection6 | googleSheets | Fetch first “sent for approval” row | set batch size2 | data allocation15 | # 4… |
| data allocation15 | set | Map fields into Shopify article create payload | get old collection6 | article creation8 | # 4… |
| article creation8 | httpRequest | Shopify REST create blog article | data allocation15 | data allocation16 | # 4… |
| data allocation16 | set | Extract admin_graphql_api_id for GraphQL lookup | article creation8 | article creation9 | # 4… |
| article creation9 | httpRequest | Shopify GraphQL query article info + domain | data allocation16 | Edit Fields | # 4… |
| Edit Fields | set | Build public article URL | article creation9 | get just added1 | # 4… |
| get just added1 | googleSheets | Update status “posted” and store article_url | Edit Fields | — | # 4… |
| Sticky Note1 | stickyNote | Documentation / overview panel | — | — | # Automation Overview: Shopify Collections to AI Blog Automation Pipeline (content preserved in canvas) |
| Sticky Note4 | stickyNote | Section label | — | — | # 1. Collect new collection data… |
| Sticky Note6 | stickyNote | Section label | — | — | ## When new data added to Store |
| Sticky Note7 | stickyNote | Section label | — | — | ## All previous collection data available in Shopify Store |
| Sticky Note10 | stickyNote | Section label | — | — | # 2. Create blog article using collection data |
| Sticky Note12 | stickyNote | Section label | — | — | ## Blog creation (One time only) |
| Sticky Note | stickyNote | Section label | — | — | # 3. Generate a relevant image for the blog article. |
| Sticky Note2 | stickyNote | Section label | — | — | # 4. Update image url to article body HTML and post to Shopify |
| Sticky Note5 | stickyNote | Author/contact panel | — | — | # Author Details / ![Manish Kumar](https://i.ibb.co/mVn8q94f/Screenshot-2026-01-18-at-8-58-23-AM.png) / manipritraj@gmail.com / +91-9334888899 |

---

## 4. Reproducing the Workflow from Scratch

1) **Create credentials**
   1. Shopify Admin API Access Token credential (n8n “Shopify Access Token API”):
      - Store domain (e.g., `your-store.myshopify.com`)
      - Admin access token with scopes for: collections read, blogs/articles write, files write (and any required GraphQL scopes).
   2. Google Sheets OAuth2 credential:
      - Authorize access to the target spreadsheet.
   3. Perplexity API credential:
      - API key.
   4. Google Gemini / PaLM credential (for n8n LangChain Gemini nodes):
      - API key and project settings as required.

2) **Create the Google Sheet**
   1. Spreadsheet: `AsTools_shopify_to_blog` (name can differ).
   2. Sheet/tab: `Collection_data`.
   3. Add columns (at minimum):
      - `id, title, handle, description, updatedAt, type, collection_url, ai_title, ai_tags, ai_description, image_prompt, image_url, article_url, status`
   4. Ensure `id` is unique and used for updates.

3) **Block: New collection intake**
   1. Add **Shopify Trigger** node:
      - Topic: `collections/create`
      - Auth: Shopify access token credential.
   2. Add **Set** node “data allocation”:
      - Map `id,title,handle,description,updatedAt` from `$json.node.*`.
   3. Add **Google Sheets** node “collection data storing1”:
      - Operation: Append
      - Map fields:
        - `id,title,handle,description,updatedAt`
        - `type = "just added"`
        - `status = "pending"`
        - `collection_url = "https://www.astools.nl/collections/{{$json.handle}}"` (recommended fix: include protocol; do not store as `=...` unless you intend a formula)
   4. Add **Wait** “2 Sec delay” (2 seconds).
   5. Add **Google Sheets** “get old collection”:
      - Filter: `type="just added"`, `status="pending"`
      - Return first match: true.
   6. Connect: Trigger → Set → Sheets Append → Wait → Sheets Get.

4) **Block: Backfill old collections (manual)**
   1. Add **Manual Trigger** “START”.
   2. Add **HTTP Request** “collection API call”:
      - POST `https://YOUR_STORE/admin/api/2025-10/graphql.json`
      - JSON body with query `collections(first: N, after: cursor)`  
      - Start with `first: 50` and add pagination loop (recommended), or keep as-is.
      - Auth: Shopify credential.
   3. Add **Split Out** “devide items”:
      - Field: `data.collections.edges`
   4. Add **Split In Batches** “batch size 10” with `batchSize=10`.
   5. Add **Set** “data allocation1” mapping `$json.node.*` to `id,title,handle,description,updatedAt`.
   6. Add **Google Sheets** “collection data storing” append:
      - Same mapping as new collections, but:
        - `type="old collections"`, `status="pending"`.
   7. Connect: Manual → HTTP → SplitOut → SplitInBatches → Set → Sheets Append.

5) **Block: Scheduled draft generation**
   1. Add **Schedule Trigger** “time 3”:
      - Configure an interval (e.g., every 10 minutes).
   2. Add **Google Sheets** “get old collection1”:
      - Filter: `type="old collections"`, `status="pending"`, returnFirstMatch=true.
   3. Add **Set** “data allocation8” to carry fields needed by AI.
   4. Add **Limit** “Limiter” (limit 1).
   5. Add **Split In Batches** “set batch size” (default batch size 1 is fine).
   6. Add **Perplexity** “web search2”:
      - Use the provided structured prompt and require JSON output.
   7. Add **LangChain components**:
      - **Gemini Chat Model** “Gemini2” with model `models/gemini-3-flash-preview` (or a stable equivalent).
      - **Memory Buffer Window** “Memory2” (use a safer session key, e.g., `{{$execution.id}}`).
      - **Structured Output Parser** “Structured2” with the article JSON schema.
      - **Gemini Chat Model** “Gemini5” for autofixing (choose a stable model).
      - **Autofixing Output Parser** “Output structure”.
      - **LangChain Agent** “content generator2”:
        - Input: product_title/description + websearch_data
        - System prompt: enforce strict JSON output.
        - Connect `Gemini2` to agent language model, `Memory2` to memory, `Output structure` to output parser.
   8. Add **Code** “HTML structuring3”:
      - Convert JSON into HTML and insert `IMAGE_REPLACE` placeholder.
   9. Add **Google Sheets Update** “update content3”:
      - Match by `id`
      - Set `status="generated"`, and store ai_title, ai_tags, ai_description (HTML), image_prompt.
   10. Connect in order:
      - schedule → get old collection1 → data allocation8 → Limiter → set batch size → web search2 → content generator2 → HTML structuring3 → update content3.

6) **Block: Scheduled one-time blog creation**
   1. Add **Schedule Trigger** “time 6” (or run manually once instead).
   2. Add **Set** “data allocation14” with title/handle.
   3. Add **HTTP Request** “article creation7” GraphQL blogCreate.
   4. Run once, then disable this trigger (recommended).

7) **Block: Image generation + upload**
   1. Add **Google Sheets Get** “get img prompt”:
      - Filter: `status="generated"` and `image_url is empty`.
   2. Add **Split In Batches** “set batch size1”.
   3. Add **HTTP Request** “create space” (GraphQL stagedUploadsCreate).
   4. Add **Gemini Image** node “image creation”:
      - Model: `models/gemini-2.5-flash-image`
      - Prompt from sheet `image_prompt`
      - Ensure output is binary field named `data`.
   5. Add **HTTP Request** “update space”:
      - multipart/form-data to `shopify-staged-uploads.storage.googleapis.com`
      - Use parameters from stagedUploadsCreate response + binary file.
   6. Add **HTTP Request** “get temp url” (GraphQL fileCreate), using `<Location>` extraction.
   7. Add **Wait** “delay1” (e.g., 10–30 seconds).
   8. Add **HTTP Request** “get img url” to query fileStatus and preview URL.
   9. Add **IF** “If verified” checking `fileStatus == READY`:
      - True → **Google Sheets Update** “Update row in sheet”: set `status="image updated"` and save `image_url`.
      - False → loop back to wait → query again (add max retry logic recommended).
   10. Connect accordingly.

8) **Block: Replace image in HTML + mark ready to publish**
   1. Add **Google Sheets Get** “get content (img updated)” filter `status="image updated"`.
   2. Add **Split In Batches** “set batch size2”.
   3. Add **Code** “replace img url” to replace `IMAGE_REPLACE` with `image_url`.
   4. Add **Google Sheets Update** “update content4” to set:
      - `status="sent for approval"`
      - `ai_description = updated body_html`.

9) **Block: Publish to Shopify and store final URL**
   1. Add **Google Sheets Get** “get old collection6”:
      - Filter: `type="old collections"`, `status="sent for approval"`, returnFirstMatch=true.
   2. Add **Set** “data allocation15” to map:
      - `blog_id` to your actual Shopify Blog numeric ID (required for REST endpoint).
      - `published_at` as ISO 8601 (recommended): `{{$now.toISO()}}`
   3. Add **HTTP Request** “article creation8” REST:
      - POST `/admin/api/2025-07/blogs/{{blog_id}}/articles.json`
   4. Add **Set** “data allocation16” extracting `article.admin_graphql_api_id`.
   5. Add **HTTP Request** “article creation9” GraphQL article query on the same store domain you use elsewhere.
   6. Add **Set** “Edit Fields” to compose public URL.
   7. Add **Google Sheets Update** “get just added1” to set:
      - `status="posted"`
      - `article_url` to computed URL.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Automation Overview panel describing the pipeline steps and setup checklist | Sticky note content in canvas |
| Author Details: Manish Kumar, Expert Designer & AI Automation Engineer, manipritraj@gmail.com, +91-9334888899 | Sticky note content |
| Author image | https://i.ibb.co/mVn8q94f/Screenshot-2026-01-18-at-8-58-23-AM.png |
| Google Sheets document used in nodes | https://docs.google.com/spreadsheets/d/1mKEh5BO9v8bY1iJNQDxu8Kvy1GMaTFheifvhCenfZOQ/edit |
| Important placeholders to replace | `{store-name}` in multiple HTTP endpoints; `{blog-id}` in publishing block |

**Key integration cautions (high impact):**
- Shopify collection backfill query is not paginated and uses `first: 2` (only two collections per manual run).
- `published_at` is not in ISO format in the current configuration.
- Two Google Sheets “Get” nodes have `executeOnce: true`, which can prevent continuous operation.
- `article creation9` uses a hardcoded domain `nl-astools.myshopify.com` (must match your store).
- Image upload parameter indexing (`parameters[0]..[8]`) is brittle; Shopify responses can change.
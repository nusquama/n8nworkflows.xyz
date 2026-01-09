Publish Google Docs to WordPress with advance RankMath SEO & Gemini analysis

https://n8nworkflows.xyz/workflows/publish-google-docs-to-wordpress-with-advance-rankmath-seo---gemini-analysis-11989


# Publish Google Docs to WordPress with advance RankMath SEO & Gemini analysis

## 1. Workflow Overview

**Purpose:** Automatically publish (as **WordPress drafts**) content written in **Google Docs** by exporting the document as HTML, cleaning/sanitizing it, generating metadata (slug, excerpt, reading time), checking for duplicates in WordPress, then using **Gemini** to produce **RankMath SEO fields** (focus keyword, SEO title, meta description, opening sentence, SEO slug). Finally, it updates the WordPress post via REST (including RankMath meta keys) and moves the processed Google Drive file into a “Published” folder.

**Typical use cases**
- Content teams writing in Google Docs and pushing drafts to WordPress with consistent formatting and SEO metadata
- Eliminating duplicate posts by matching on slug
- Automating RankMath SEO fields (via custom WP helper snippet as noted)

### 1.1 Input Reception & Global Configuration
Triggered on new files in a specific Drive folder; loads workflow configuration (WP base URL, Published folder ID).

### 1.2 Export & HTML Sanitization
Exports the Google Doc to HTML via Google Drive API, extracts `<body>`, removes unwanted styles/attributes/wrappers, decodes entities.

### 1.3 Metadata Preparation
Builds plain-text version, excerpt, Vietnamese-friendly slug, word count and reading time.

### 1.4 Duplicate Check (Create vs Update)
Searches WP posts by slug; if found updates existing draft, otherwise creates a new draft.

### 1.5 AI SEO Analysis (Gemini) + QA
Sends title/excerpt to Gemini to get a **strict JSON** SEO payload; parses/validates it and prepares RankMath meta + patched content.

### 1.6 WordPress SEO Update, Verification & Drive Archiving
PATCHes WP post with RankMath meta and updated content, verifies result, then moves Drive file to “Published”.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & Global Configuration
**Overview:** Watches a Drive folder for new files and injects required settings used by later nodes (WordPress base URL and destination “Published” folder ID).

**Nodes involved**
- Google Drive Trigger
- CONFIG - Edit Settings Here

#### Node: Google Drive Trigger
- **Type / role:** `googleDriveTrigger` — Entry point; polls for new files.
- **Key configuration (interpreted):**
  - Event: `fileCreated`
  - Trigger scope: `specificFolder` (must be selected in node UI; JSON shows empty value)
  - Polling: every minute
- **Inputs/outputs:**
  - **Output →** `CONFIG - Edit Settings Here`
- **Credentials:** Google Drive OAuth2
- **Edge cases / failures:**
  - Trigger folder not set ⇒ no events
  - Drive permission issues (shared drives, missing access)
  - Polling duplicates (rare) if Drive reports same file multiple times—downstream slug check mitigates WP duplication, but Drive move could fail if file already moved.

#### Node: CONFIG - Edit Settings Here
- **Type / role:** `set` — Centralized config values.
- **Key configuration:**
  - `wp_base_url`: placeholder “Input your domain here” (must be set to e.g. `https://example.com`)
  - `drive_published_folder_id`: placeholder (must be a valid folder ID)
- **Inputs/outputs:**
  - **Input ←** Google Drive Trigger
  - **Output →** `HTTP Request (Export HTML)`
- **Edge cases / failures:**
  - Invalid `wp_base_url` leads to failing WP HTTP calls (DNS/404/SSL).
  - Wrong folder ID causes final move to fail.

---

### Block 2 — Export & HTML Sanitization
**Overview:** Exports the Google Doc to HTML via Drive API, extracts the body content, strips styling/attributes, removes span wrappers, and decodes common HTML entities.

**Nodes involved**
- HTTP Request (Export HTML)
- Code (Extract Body)

#### Node: HTTP Request (Export HTML)
- **Type / role:** `httpRequest` — Calls Google Drive “export” endpoint.
- **Key configuration:**
  - URL: `https://www.googleapis.com/drive/v3/files/{{DriveFileId}}/export`
  - Query parameters:
    - `supportsAllDrives=true`
    - `mimeType=text/html`
  - Auth: predefined credential type `googleDriveOAuth2Api`
- **Inputs/outputs:**
  - **Input ←** CONFIG
  - **Uses expression:** file id from `Google Drive Trigger` (`$('Google Drive Trigger').item.json.id`)
  - **Output →** `Code (Extract Body)`
- **Edge cases / failures:**
  - Non-Google-Docs file types may not support export to HTML ⇒ 400/403
  - Large docs: response size/timeouts
  - If Google returns non-HTML or different payload shape, downstream extraction may fail

#### Node: Code (Extract Body)
- **Type / role:** `code` — Cleans the HTML string and emits normalized fields.
- **Key configuration choices:**
  - Extracts `<body>…</body>` if present; otherwise uses full response.
  - Removes `<style>`, `<meta>`, and outer html/head/body tags.
  - Strips many attributes (style/class/id/etc + data-* + aria-*)
  - Repeatedly unwraps `<span>` tags (up to 5 passes)
  - Removes empty `<p>` and normalizes `<hr>`
  - Decodes numeric entities and a mapping of named entities (quotes, dashes, Latin-1 accents)
  - Final whitespace normalization
- **Output JSON:**
  - `fileId`, `title` (Drive file name), `htmlBody`, `bodyLength`
- **Inputs/outputs:**
  - **Input ←** HTTP Request (Export HTML)
  - **Output →** `Code (Prepare Metadata)`
- **Edge cases / failures:**
  - If `HTTP Request (Export HTML)` returns binary/other structure, `raw.match(...)` may error or produce empty body.
  - Aggressive attribute stripping can remove meaningful attributes (e.g., `href` is preserved, but any unforeseen attr needed for embeds could be removed).
  - Entity decoding map is partial; uncommon entities may remain.

---

### Block 3 — Metadata Preparation
**Overview:** Produces derived metadata: plain text, excerpt, slug (accent-insensitive, Vietnamese-friendly), word count, and reading time.

**Nodes involved**
- Code (Prepare Metadata)

#### Node: Code (Prepare Metadata)
- **Type / role:** `code` — Converts HTML to text, computes stats, generates excerpt and slug.
- **Key configuration choices:**
  - Normalizes `<hr>` to `<hr />`
  - `htmlToText()` strips tags, converts block closings to newlines, converts `<hr>` to `---`
  - Word count by splitting on whitespace
  - Reading time: `max(1, round(words/200))`
  - Excerpt: ~180 chars, word-safe truncation with ellipsis
  - Slug: accent removal + `đ`→`d` + non-alphanumerics to hyphens
- **Output JSON:**
  - `fileId`, `title`, `slug`, `excerpt`, `htmlBody`, `textBody`, `wordCount`, `readingTimeMin`, `updatedAt`
- **Inputs/outputs:**
  - **Input ←** Code (Extract Body)
  - **Outputs →**
    - `HTTP Request (WP - Find Post by Slug)`
    - `Merge (Meta + FindResult)` (input 0)
- **Edge cases / failures:**
  - Empty title ⇒ empty slug ⇒ WP search less reliable / might match nothing; WP create may fail or auto-generate.
  - Very short content produces minimal excerpt and low word count.
  - Slug collisions across posts will result in updating/creating behavior based solely on slug.

---

### Block 4 — Duplicate Check (Create vs Update)
**Overview:** Searches WordPress for an existing post with the same slug. If found, updates it as a draft; if not found, creates a new draft.

**Nodes involved**
- HTTP Request (WP - Find Post by Slug)
- Merge (Meta + FindResult)
- IF (Found Post?)
- HTTP Request - WP Update Post
- HTTP Request: WP - Create Post

#### Node: HTTP Request (WP - Find Post by Slug)
- **Type / role:** `httpRequest` — WordPress REST API query to detect duplicates.
- **Key configuration:**
  - GET `{{wp_base_url}}/wp-json/wp/v2/posts`
  - Query: `slug={{$json.slug}}`, `per_page=1`, `status=any`
  - Response: fullResponse + JSON
  - `alwaysOutputData=true` (so workflow continues even on some failures)
- **Inputs/outputs:**
  - **Input ←** Code (Prepare Metadata)
  - **Output →** Merge (Meta + FindResult) (input 1)
- **Credentials:** WordPress API credential (application password or OAuth depending on n8n setup)
- **Edge cases / failures:**
  - Auth failure (401/403) ⇒ merge/IF logic may mis-detect “not found” and create duplicates.
  - If WP returns unexpected shape, `IF (Found Post?)` condition may behave incorrectly.
  - Using `status=any` requires permissions to read private/draft posts.

#### Node: Merge (Meta + FindResult)
- **Type / role:** `merge` (combineAll) — Combines metadata item with WP find result item.
- **Configuration:** `mode=combine`, `combineBy=combineAll` (effectively merges the two incoming item JSONs into one item)
- **Inputs/outputs:**
  - **Input 0 ←** Code (Prepare Metadata)
  - **Input 1 ←** HTTP Request (WP - Find Post by Slug)
  - **Output →** IF (Found Post?)
- **Edge cases / failures:**
  - If one side produces multiple items, combineAll can create Cartesian merges. Here both are expected to be single-item.

#### Node: IF (Found Post?)
- **Type / role:** `if` — Branches on whether WP returned at least one post.
- **Condition:**
  - `{{ $json.body?.length ?? 0 }} > 0`
  - Assumes the WP find node populates `body` array (because fullResponse is enabled).
- **Outputs:**
  - **True →** HTTP Request - WP Update Post
  - **False →** HTTP Request: WP - Create Post
- **Edge cases / failures:**
  - If WP call failed but still outputs data, `body` might be undefined ⇒ treated as not found ⇒ creates new post unintentionally.

#### Node: HTTP Request - WP Update Post
- **Type / role:** `httpRequest` — Updates existing WP post (draft).
- **Key configuration:**
  - PUT `{{wp_base_url}}/wp-json/wp/v2/posts/{{$json.body[0].id}}`
  - Body: `title`, `slug`, `status=draft`, `content={{htmlBody}}`, `excerpt={{excerpt}}`
- **Inputs/outputs:**
  - **Input ←** IF true branch
  - **Output →** SET - SEO Input
- **Edge cases / failures:**
  - If `body[0].id` missing ⇒ URL invalid
  - PUT may require higher privileges; may fail on locked posts/revisions/plugins

#### Node: HTTP Request: WP - Create Post
- **Type / role:** `httpRequest` — Creates new WP post (draft).
- **Key configuration:**
  - POST `{{wp_base_url}}/wp-json/wp/v2/posts`
  - Body: `title`, `slug`, `status=draft`, `content`, `excerpt`
- **Inputs/outputs:**
  - **Input ←** IF false branch
  - **Output →** SET - SEO Input
- **Edge cases / failures:**
  - Duplicate slug may cause WP to auto-modify slug (e.g., adding `-2`), which can desync later “find by slug” assumptions (this workflow continues using the slug it set, plus it also sets slug again later).

---

### Block 5 — AI SEO Analysis (Gemini) + QA + Payload Build
**Overview:** Standardizes the postId/title/excerpt/content fields, requests Gemini to generate SEO JSON, parses and validates it, then builds the RankMath meta payload and patches the HTML to inject the focus keyword early.

**Nodes involved**
- SET - SEO Input
- GEMINI - SEO JSON
- MERGE - Attach postId
- CODE - Parse & QA SEO
- CODE - Build WP Payload

#### Node: SET - SEO Input
- **Type / role:** `set` — Normalizes the fields needed by Gemini and later merges.
- **Key assignments / expressions:**
  - `postId`: `{{$json.id || $json.body?.[0]?.id}}` (works for both create and update paths)
  - `title`: prefers `title.raw` then fallback `title/name`
  - `excerpt.raw`: prefers `excerpt.raw` else `excerpt`
  - `htmlBody`, `textBody`, `fileId`: pulled from `Code (Prepare Metadata)`
- **Outputs:**
  - **To GEMINI - SEO JSON**
  - **To MERGE - Attach postId** (input 0)
- **Edge cases / failures:**
  - If WP response doesn’t include `id` or `body[0].id`, downstream will fail with “missing postId”.
  - If Drive trigger fires for a file without name/title, slug/title may become empty.

#### Node: GEMINI - SEO JSON
- **Type / role:** `@n8n/n8n-nodes-langchain.googleGemini` — Requests Gemini model output.
- **Key configuration:**
  - Model: `models/gemini-2.5-flash`
  - Prompt requests **JSON-only** with: `seo_title`, `meta_description`, `focus_keyword`, `opening_sentence`, `slug`, `postid`
  - Uses: title, postId, excerpt, base URL
  - `maxTries=5`, wait 5s between tries; node-level retryOnFail disabled (retries handled by node)
- **Inputs/outputs:**
  - **Input ←** SET - SEO Input
  - **Output →** MERGE - Attach postId (input 1)
- **Credentials:** `googlePalmApi` (Gemini/PaLM credential in n8n)
- **Edge cases / failures:**
  - Model may return non-JSON or extra text; downstream parser tries to recover.
  - Output JSON key mismatch: prompt uses `"postid"` but downstream expects `postId` from merge, not from model.
  - Quota/rate limits; content safety blocks; transient 5xx.

#### Node: MERGE - Attach postId
- **Type / role:** `merge` (combineAll) — Joins Gemini output with original `postId/htmlBody/textBody`.
- **Inputs/outputs:**
  - **Input 0 ←** SET - SEO Input
  - **Input 1 ←** GEMINI - SEO JSON
  - **Output →** CODE - Parse & QA SEO
- **Edge cases / failures:**
  - If Gemini returns multiple items unexpectedly, combineAll may create multiple merged outputs.

#### Node: CODE - Parse & QA SEO
- **Type / role:** `code` — Extracts JSON from Gemini response and validates required identifiers.
- **Key logic:**
  - `extractJson()` parses:
    - fenced ```json blocks
    - generic ``` blocks
    - otherwise attempts substring from first `{` to last `}`
  - Validates `postId` is finite and > 0; throws error if missing
  - Reads Gemini text from `j.content?.parts?.[0]?.text` (LangChain Gemini node output format)
  - Outputs: `postId`, `seo_title`, `meta_description`, `focus_keyword`, `slug`, `_debug_rawText`, `htmlBody`, `textBody`
- **Inputs/outputs:**
  - **Input ←** MERGE - Attach postId
  - **Output →** CODE - Build WP Payload
- **Edge cases / failures:**
  - If Gemini node output schema changes (e.g., different field path than `content.parts[0].text`), parser will treat raw as empty and produce empty SEO fields.
  - If Gemini returns invalid JSON with unmatched braces, substring parse can still throw.

#### Node: CODE - Build WP Payload
- **Type / role:** `code` — Builds WP PATCH payload fields and injects keyword marker.
- **Key logic:**
  - Trims/clamps:
    - `seo_title` to 70 chars (even though prompt says max 60)
    - `meta_description` to 170 chars
  - Builds `meta` object:
    - `rank_math_title`
    - `rank_math_description`
    - `rank_math_focus_keyword`
    - Removes empty meta keys
  - `injectKeywordToHtml()`:
    - Adds marker `<!--n8n_kw_injected:...-->` to avoid duplicate injections
    - Checks if keyword already appears early in content; if not, injects `<strong>{keyword}</strong>:` at first `<p>` or prepends a new `<p>`
  - Outputs: `postId`, `slug`, `meta`, `contentPatched`, `_seo`
- **Inputs/outputs:**
  - **Input ←** CODE - Parse & QA SEO
  - **Output →** HTTP - WP Update SEO (RankMath)
- **Edge cases / failures:**
  - If HTML starts with a non-paragraph structure (e.g., heading), injection may still modify first `<p>` later, not truly “opening sentence”.
  - Marker prevents repeated injections across re-runs, but only if the stored WP content still contains the marker.

---

### Block 6 — WordPress SEO Update, Verification & Drive Archiving
**Overview:** PATCHes the WordPress post to apply slug, excerpt, RankMath meta fields, and patched content; then fetches the post for verification and moves the Drive file to the Published folder.

**Nodes involved**
- HTTP - WP Update SEO (RankMath)
- HTTP - WP Get Post (Verify)
- Google Drive - Move to Published

#### Node: HTTP - WP Update SEO (RankMath)
- **Type / role:** `httpRequest` — Applies final post fields and RankMath meta via REST API.
- **Key configuration:**
  - PATCH `{{wp_base_url}}/wp-json/wp/v2/posts/{{$json.postId}}`
  - Body parameters:
    - `status=draft`
    - `slug={{$json.slug}}`
    - `excerpt={{ $('SET - SEO Input').item.json.excerpt.raw }}`
    - `meta={{$json.meta}}` (RankMath keys)
    - `content={{$json.contentPatched}}`
- **Inputs/outputs:**
  - **Input ←** CODE - Build WP Payload
  - **Output →** HTTP - WP Get Post (Verify)
- **Sticky note requirement:** “RankMath Helper PHP Snippet” must be installed.
- **Edge cases / failures:**
  - Core WordPress REST API may block updating arbitrary `meta` unless:
    - meta keys are registered with `show_in_rest`, or
    - a plugin/snippet exposes them (this is the “critical requirement”)
  - PATCH method may be rejected on some hosts; switching to POST with `_method` or PUT may be needed.
  - If WP sanitizes/removes HTML marker comments, injection marker may be lost.

#### Node: HTTP - WP Get Post (Verify)
- **Type / role:** `httpRequest` — Fetches updated post for verification/debug.
- **Key configuration:**
  - GET `{{wp_base_url}}/wp-json/wp/v2/posts/{{ $json.id }}?context=edit`
- **Inputs/outputs:**
  - **Input ←** HTTP - WP Update SEO (RankMath)
  - **Output →** Google Drive - Move to Published
- **Potential issue:** This node references `{{$json.id}}`, but upstream payload outputs `postId` and WP PATCH typically returns the post object including `id`. If WP returns a different schema or the node output is altered, `id` may be missing.
- **Edge cases / failures:**
  - `context=edit` requires authentication with edit permissions
  - If `id` is missing, request URL becomes invalid

#### Node: Google Drive - Move to Published
- **Type / role:** `googleDrive` — Moves the processed source file to an archive folder.
- **Key configuration:**
  - Operation: `move`
  - `fileId`: from `SET - SEO Input` (`fileId`)
  - Destination folder: `drive_published_folder_id` from CONFIG
- **Inputs/outputs:**
  - **Input ←** HTTP - WP Get Post (Verify)
  - Terminal node (no outputs)
- **Edge cases / failures:**
  - If file already moved/deleted, move fails
  - If destination folder is in another drive/shared drive without permission, move fails

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Google Drive Trigger | googleDriveTrigger | Watches a Drive folder for new files | — | CONFIG - Edit Settings Here | ## 1. Input & Cleaning<br>Monitors Google Drive for new files, sanitizes raw HTML, and prepares metadata for processing. |
| CONFIG - Edit Settings Here | set | Stores WP base URL + Published folder ID | Google Drive Trigger | HTTP Request (Export HTML) | ## 1. Input & Cleaning<br>Monitors Google Drive for new files, sanitizes raw HTML, and prepares metadata for processing. |
| HTTP Request (Export HTML) | httpRequest | Export Google Doc as HTML via Drive API | CONFIG - Edit Settings Here | Code (Extract Body) | ## 1. Input & Cleaning<br>Monitors Google Drive for new files, sanitizes raw HTML, and prepares metadata for processing. |
| Code (Extract Body) | code | Extracts `<body>`, strips styles/attributes, decodes entities | HTTP Request (Export HTML) | Code (Prepare Metadata) | ## 1. Input & Cleaning<br>Monitors Google Drive for new files, sanitizes raw HTML, and prepares metadata for processing. |
| Code (Prepare Metadata) | code | Builds excerpt/slug/text, word count, reading time | Code (Extract Body) | HTTP Request (WP - Find Post by Slug); Merge (Meta + FindResult) | ## 1. Input & Cleaning<br>Monitors Google Drive for new files, sanitizes raw HTML, and prepares metadata for processing. |
| HTTP Request (WP - Find Post by Slug) | httpRequest | Checks if post exists by slug | Code (Prepare Metadata) | Merge (Meta + FindResult) | ## 2. Duplicate Check<br>Queries WordPress to check if the post exists. Routes to "Create" or "Update" path to prevent duplicates. |
| Merge (Meta + FindResult) | merge | Combines metadata + WP search response | Code (Prepare Metadata); HTTP Request (WP - Find Post by Slug) | IF (Found Post?) | ## 2. Duplicate Check<br>Queries WordPress to check if the post exists. Routes to "Create" or "Update" path to prevent duplicates. |
| IF (Found Post?) | if | Routes to Update or Create | Merge (Meta + FindResult) | HTTP Request - WP Update Post; HTTP Request: WP - Create Post | ## 2. Duplicate Check<br>Queries WordPress to check if the post exists. Routes to "Create" or "Update" path to prevent duplicates. |
| HTTP Request - WP Update Post | httpRequest | Updates existing WP post as draft | IF (Found Post?) (true) | SET - SEO Input | ## 2. Duplicate Check<br>Queries WordPress to check if the post exists. Routes to "Create" or "Update" path to prevent duplicates. |
| HTTP Request: WP - Create Post | httpRequest | Creates new WP post as draft | IF (Found Post?) (false) | SET - SEO Input | ## 2. Duplicate Check<br>Queries WordPress to check if the post exists. Routes to "Create" or "Update" path to prevent duplicates. |
| SET - SEO Input | set | Normalizes postId/title/excerpt/html/text for AI & downstream | HTTP Request - WP Update Post; HTTP Request: WP - Create Post | GEMINI - SEO JSON; MERGE - Attach postId | ## 3. AI SEO Analysis<br>Gemini analyzes content to generate Focus Keyword, SEO Title, and Meta Description in structured JSON. |
| GEMINI - SEO JSON | googleGemini (LangChain) | Generates SEO JSON (focus keyword/title/meta/slug/opening sentence) | SET - SEO Input | MERGE - Attach postId | ## 3. AI SEO Analysis<br>Gemini analyzes content to generate Focus Keyword, SEO Title, and Meta Description in structured JSON. |
| MERGE - Attach postId | merge | Combines SET fields with Gemini output | SET - SEO Input; GEMINI - SEO JSON | CODE - Parse & QA SEO | ## 3. AI SEO Analysis<br>Gemini analyzes content to generate Focus Keyword, SEO Title, and Meta Description in structured JSON. |
| CODE - Parse & QA SEO | code | Extracts JSON from Gemini text; validates postId | MERGE - Attach postId | CODE - Build WP Payload | ## 3. AI SEO Analysis<br>Gemini analyzes content to generate Focus Keyword, SEO Title, and Meta Description in structured JSON. |
| CODE - Build WP Payload | code | Builds RankMath meta + patched content (keyword injection) | CODE - Parse & QA SEO | HTTP - WP Update SEO (RankMath) | ## 3. AI SEO Analysis<br>Gemini analyzes content to generate Focus Keyword, SEO Title, and Meta Description in structured JSON. |
| HTTP - WP Update SEO (RankMath) | httpRequest | PATCH post with slug/content/excerpt + RankMath meta | CODE - Build WP Payload | HTTP - WP Get Post (Verify) | ## 4. Publish & Cleanup<br>Updates WordPress content, maps custom RankMath meta keys via API, and moves files to the "Published" folder.<br>### ⚠️ CRITICAL REQUIREMENT<br>This node requires the **RankMath Helper PHP Snippet** to be installed on your WordPress site.<br>Without it, RankMath fields cannot be updated via API. Check the template description! |
| HTTP - WP Get Post (Verify) | httpRequest | Fetches updated post (context=edit) | HTTP - WP Update SEO (RankMath) | Google Drive - Move to Published | ## 4. Publish & Cleanup<br>Updates WordPress content, maps custom RankMath meta keys via API, and moves files to the "Published" folder. |
| Google Drive - Move to Published | googleDrive | Moves processed file to Published folder | HTTP - WP Get Post (Verify) | — | ## 4. Publish & Cleanup<br>Updates WordPress content, maps custom RankMath meta keys via API, and moves files to the "Published" folder. |
| Sticky Note | stickyNote | Comment container | — | — | ## 1. Input & Cleaning<br>Monitors Google Drive for new files, sanitizes raw HTML, and prepares metadata for processing. |
| Sticky Note1 | stickyNote | Comment container | — | — | ## 2. Duplicate Check<br>Queries WordPress to check if the post exists. Routes to "Create" or "Update" path to prevent duplicates. |
| Sticky Note2 | stickyNote | Comment container | — | — | ## 3. AI SEO Analysis<br>Gemini analyzes content to generate Focus Keyword, SEO Title, and Meta Description in structured JSON. |
| Sticky Note3 | stickyNote | Comment container | — | — | ## 4. Publish & Cleanup<br>Updates WordPress content, maps custom RankMath meta keys via API, and moves files to the "Published" folder. |
| Sticky Note4 | stickyNote | Comment container | — | — | # Google Drive to WordPress Publisher<br>**Auto-publish content with Advanced RankMath SEO & Gemini Analysis.**<br><br>This workflow automates the "Document Ops" process for professional content teams. It creates a seamless bridge between writing in Google Docs and publishing on WordPress, ensuring high-quality SEO standards without manual effort.<br><br>### How it works<br>1. **Ingestion:** The workflow monitors a specific Google Drive folder for new HTML or Doc files.<br>2. **Sanitization:** It cleans the raw HTML from Google Docs, removing messy styles and tags to ensure clean code.<br>3. **Smart Logic:** It checks if the post already exists (via slug) to decide whether to Create a new draft or Update an existing one.<br>4. **AI SEO Agent:** Gemini AI analyzes the content to extract the Focus Keyword, SEO Title, and Meta Description, optimizing for RankMath.<br>5. **Publishing:** The content is pushed to WordPress via REST API, mapping custom RankMath meta keys that standard nodes often miss.<br>6. **Archiving:** Processed files are moved to a "Published" folder.<br><br>### Setup steps<br>1. **Configure Credentials:** Set up your Google Drive, Gemini (PaLM), and WordPress credentials.<br>2. **Global Config:** In the `CONFIG` node, enter your WordPress URL and the ID of your "Published" folder in Drive.<br>3. **RankMath API Helper:** You **MUST** install the custom PHP snippet provided in the template description to allow n8n to update RankMath metadata (Critical Step).<br>4. **Trigger:** Select your "Drafts" folder in the Google Drive Trigger node. |
| Sticky Note5 | stickyNote | Comment container | — | — | ### ⚠️ CRITICAL REQUIREMENT<br>This node requires the **RankMath Helper PHP Snippet** to be installed on your WordPress site.<br>Without it, RankMath fields cannot be updated via API. Check the template description! |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Trigger: “Google Drive Trigger”**
   - Node type: *Google Drive Trigger*
   - Event: **File Created**
   - Trigger on: **Specific folder**
   - Choose the “Drafts” folder to watch
   - Polling: every minute
   - Credential: **Google Drive OAuth2** (account must have access to the folder)

2. **Add config node: “CONFIG - Edit Settings Here”**
   - Node type: *Set*
   - Add fields:
     - `wp_base_url` (string): e.g. `https://yourdomain.com`
     - `drive_published_folder_id` (string): the destination folder ID in Drive
   - Connect: **Google Drive Trigger → CONFIG**

3. **Export Doc as HTML: “HTTP Request (Export HTML)”**
   - Node type: *HTTP Request*
   - Method: **GET**
   - URL: `https://www.googleapis.com/drive/v3/files/{{FILE_ID}}/export`
     - FILE_ID expression: `{{ $('Google Drive Trigger').item.json.id }}`
   - Authentication: **Google Drive OAuth2 (predefined credential)**
   - Query params:
     - `supportsAllDrives=true`
     - `mimeType=text/html`
   - Connect: **CONFIG → HTTP Request (Export HTML)**

4. **Clean HTML: “Code (Extract Body)”**
   - Node type: *Code*
   - Paste the provided logic to:
     - extract `<body>`
     - remove styles/meta/outer tags
     - strip attributes
     - unwrap spans
     - decode entities
   - Ensure outputs:
     - `fileId`, `title`, `htmlBody`, `bodyLength`
   - Connect: **HTTP Request (Export HTML) → Code (Extract Body)**

5. **Prepare metadata: “Code (Prepare Metadata)”**
   - Node type: *Code*
   - Implement:
     - `htmlToText()`
     - `wordCount`, `readingTimeMin`
     - `excerpt` (≈180 chars)
     - `slug` (accent-insensitive, Vietnamese-friendly)
   - Output: `title`, `slug`, `excerpt`, `htmlBody`, `textBody`, `fileId`, etc.
   - Connect: **Code (Extract Body) → Code (Prepare Metadata)**

6. **Find existing post: “HTTP Request (WP - Find Post by Slug)”**
   - Node type: *HTTP Request*
   - Method: **GET**
   - URL: `{{wp_base_url}}/wp-json/wp/v2/posts`
   - Query params:
     - `slug={{$json.slug}}`
     - `per_page=1`
     - `status=any`
   - Response: enable **Full Response** (so you get `body`)
   - Credential: **WordPress API**
   - Connect: **Code (Prepare Metadata) → HTTP Request (WP - Find Post by Slug)**

7. **Merge metadata + find result: “Merge (Meta + FindResult)”**
   - Node type: *Merge*
   - Mode: **Combine**
   - Combine by: **Combine All**
   - Connect:
     - **Code (Prepare Metadata) → Merge (input 0)**
     - **HTTP Request (WP - Find Post by Slug) → Merge (input 1)**

8. **Branch: “IF (Found Post?)”**
   - Node type: *IF*
   - Condition (Number): `{{ $json.body?.length ?? 0 }}` **greater than** `0`
   - Connect: **Merge → IF**

9. **Update path: “HTTP Request - WP Update Post” (IF true)**
   - Node type: *HTTP Request*
   - Method: **PUT**
   - URL: `{{wp_base_url}}/wp-json/wp/v2/posts/{{$json.body[0].id}}`
   - Body (JSON):
     - `title={{$json.title}}`
     - `slug={{$json.slug}}`
     - `status=draft`
     - `content={{$json.htmlBody}}`
     - `excerpt={{$json.excerpt}}`
   - Credential: **WordPress API**
   - Connect: **IF true → WP Update Post**

10. **Create path: “HTTP Request: WP - Create Post” (IF false)**
    - Node type: *HTTP Request*
    - Method: **POST**
    - URL: `{{wp_base_url}}/wp-json/wp/v2/posts`
    - Body (JSON): same fields as update (status draft)
    - Connect: **IF false → WP Create Post**

11. **Normalize for AI: “SET - SEO Input”**
    - Node type: *Set*
    - Fields:
      - `postId` (number): `{{$json.id || $json.body?.[0]?.id}}`
      - `title` (string): `{{$json.title?.raw || $json.title || $json.name || ''}}`
      - `excerpt.raw` (string): `{{$json.excerpt?.raw || $json.excerpt || ''}}`
      - `htmlBody` (string): from `Code (Prepare Metadata)`
      - `textBody` (string): from `Code (Prepare Metadata)`
      - `fileId` (string): from `Code (Prepare Metadata)`
    - Connect:
      - **WP Update Post → SET - SEO Input**
      - **WP Create Post → SET - SEO Input**

12. **Gemini call: “GEMINI - SEO JSON”**
    - Node type: *Google Gemini (LangChain)*
    - Model: `models/gemini-2.5-flash`
    - Prompt: require **JSON-only** output with `seo_title`, `meta_description`, `focus_keyword`, `opening_sentence`, `slug`, `postid`
    - Credential: **Google PaLM/Gemini**
    - Connect: **SET - SEO Input → GEMINI - SEO JSON**

13. **Merge Gemini + base fields: “MERGE - Attach postId”**
    - Node type: *Merge*
    - Mode: **Combine** / **Combine All**
    - Connect:
      - **SET - SEO Input → MERGE (input 0)**
      - **GEMINI - SEO JSON → MERGE (input 1)**

14. **Parse & validate: “CODE - Parse & QA SEO”**
    - Node type: *Code*
    - Implement robust JSON extraction from Gemini text:
      - read text from `content.parts[0].text`
      - parse fenced block or `{...}` substring
      - validate `postId`
    - Output: `postId`, `seo_title`, `meta_description`, `focus_keyword`, `slug`, `htmlBody`, `textBody`, `_debug_rawText`
    - Connect: **MERGE - Attach postId → CODE - Parse & QA SEO**

15. **Build WP PATCH payload: “CODE - Build WP Payload”**
    - Node type: *Code*
    - Build:
      - `meta.rank_math_title`, `meta.rank_math_description`, `meta.rank_math_focus_keyword`
      - `contentPatched` (inject keyword early with a marker comment)
    - Connect: **CODE - Parse & QA SEO → CODE - Build WP Payload**

16. **Apply RankMath meta via REST: “HTTP - WP Update SEO (RankMath)”**
    - Node type: *HTTP Request*
    - Method: **PATCH**
    - URL: `{{wp_base_url}}/wp-json/wp/v2/posts/{{$json.postId}}`
    - Body:
      - `status=draft`
      - `slug={{$json.slug}}`
      - `excerpt={{ $('SET - SEO Input').item.json.excerpt.raw }}`
      - `meta={{$json.meta}}`
      - `content={{$json.contentPatched}}`
    - Credential: **WordPress API**
    - **Important:** Install the **RankMath Helper PHP snippet** (or otherwise expose these meta keys to REST).
    - Connect: **CODE - Build WP Payload → HTTP - WP Update SEO (RankMath)**

17. **Verify: “HTTP - WP Get Post (Verify)”**
    - Node type: *HTTP Request*
    - Method: **GET**
    - URL: `{{wp_base_url}}/wp-json/wp/v2/posts/{{ $json.id }}?context=edit`
    - Credential: **WordPress API**
    - Connect: **HTTP - WP Update SEO (RankMath) → HTTP - WP Get Post (Verify)**

18. **Archive source file: “Google Drive - Move to Published”**
    - Node type: *Google Drive*
    - Operation: **Move**
    - File ID: `{{ $('SET - SEO Input').item.json.fileId }}`
    - Destination folder ID: `{{ $('CONFIG - Edit Settings Here').item.json.drive_published_folder_id }}`
    - Credential: **Google Drive OAuth2**
    - Connect: **HTTP - WP Get Post (Verify) → Google Drive - Move to Published**

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques. | Disclaimer provided by user |
| “RankMath API Helper: You MUST install the custom PHP snippet provided in the template description to allow n8n to update RankMath metadata (Critical Step).” | Required for `HTTP - WP Update SEO (RankMath)` to successfully write RankMath meta keys |
| Trigger setup reminder: select your “Drafts” folder in the Google Drive Trigger node. | Prevents silent non-triggering |
| Workflow operates as **draft-first** publishing (status always `draft`). | Change status to `publish` if desired, but ensure review process and permissions |
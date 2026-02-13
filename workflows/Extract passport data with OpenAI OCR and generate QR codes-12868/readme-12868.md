Extract passport data with OpenAI OCR and generate QR codes

https://n8nworkflows.xyz/workflows/extract-passport-data-with-openai-ocr-and-generate-qr-codes-12868


# Extract passport data with OpenAI OCR and generate QR codes

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Extract passport data with OpenAI OCR and generate QR codes  
**Workflow name (n8n):** Extract passport data with OpenAI and create QR codes

**Purpose:**  
Collect multiple passport image uploads via an n8n Form, preprocess each image, call the OpenAI Responses API to extract structured passport fields (OCR + parsing), standardize/validate the extracted data, generate a QR-code URL per passport, then aggregate results and deliver them through the form completion screen and via Gmail.

**Target use cases:**
- Batch processing of passport scans/photos to produce standardized identity fields
- Generating QR codes containing a tab-delimited payload for downstream systems
- Rapid data capture workflows with human-in-the-loop upload

### 1.1 Logical Blocks

1. **Form input processing (entry & split)**
   - Receives multiple uploaded files and converts them into individual items to process sequentially.

2. **Image preprocessing**
   - Validates file extension is an image, resizes, converts to Base64, and prepares payload for OCR.

3. **OpenAI OCR extraction**
   - Sends image + extraction instructions to OpenAI Responses endpoint.

4. **Data standardization & validation**
   - Extracts JSON from the model output, validates required fields, normalizes dates, generates derived fields, and determines passport status.

5. **QR creation & result aggregation**
   - Builds a QR payload + URL, aggregates resulting QR images, then returns HTML to the form completion page and emails it.

---

## 2. Block-by-Block Analysis

### Block A — Form input processing (entry & split)

**Overview:**  
Accepts multiple uploaded passport images from an n8n Form trigger and converts the multi-file binary payload into a list of single-file items for batch/loop processing.

**Nodes involved:**
- On form submission
- Prepare form
- Loop Over images

#### Node: On form submission
- **Type / role:** `Form Trigger` (n8n form entry point)
- **Configuration choices:**
  - Form title: *Process passport images*
  - Description: *Upload multiple passport images and generate QR codes*
  - Field: **Passport images** (file upload, required)
- **Inputs / outputs:**
  - **Output:** One execution item containing multiple binaries (one per uploaded file) under `binary`.
  - Connects to **Prepare form**.
- **Version requirements:** typeVersion **2.2**
- **Potential failures / edge cases:**
  - Large uploads can exceed instance limits (n8n binary size limits, reverse proxy limits).
  - Non-image files can be uploaded (handled later by extension check).

#### Node: Prepare form
- **Type / role:** `Code` node (normalize uploaded files into per-item structure)
- **Configuration choices:**
  - Reads `items[0].binary` and iterates all keys (each uploaded file becomes one output item).
  - Output item structure:
    - `binary.data` = the file binary
    - `json.fileName`, `json.mimeType` copied from binary metadata
- **Key variables/expressions:**
  - Uses `items[0].binary` (implicit Code node input).
- **Inputs / outputs:**
  - **Input:** single item from Form Trigger with multi-binary payload
  - **Output:** multiple items, one per uploaded file
  - Connects to **Loop Over images**
- **Version requirements:** typeVersion **2**
- **Potential failures / edge cases:**
  - If the trigger payload structure changes (e.g., no binaries), loop yields empty output.
  - Very large binaries may impact memory.

#### Node: Loop Over images
- **Type / role:** `Split In Batches` (sequential looping)
- **Configuration choices:**
  - Uses `options.reset` expression: `={{ $prevNode.name == 'On form submission'}}`
    - Resets batching when a new form submission starts.
- **Inputs / outputs:**
  - Receives multiple items from **Prepare form**
  - **Main output (index 0)** goes to:
    - **Grab image** (aggregation path)
    - **If is image** (processing path)
  - Additional loop-backs exist from later nodes back to this node to continue iteration.
- **Version requirements:** typeVersion **3**
- **Potential failures / edge cases:**
  - Reset expression depends on node naming; renaming nodes can break reset behavior.
  - If downstream nodes return no items, loop may not behave as expected.

---

### Block B — Image preprocessing

**Overview:**  
Ensures each file is an image (by extension), resizes it for OCR reliability, converts binary to Base64, and attaches `imageBase64` to the item used for the OpenAI request.

**Nodes involved:**
- If is image
- Resize Image
- Update imageBase64

#### Node: If is image
- **Type / role:** `IF` (file extension validation)
- **Configuration choices:**
  - Condition: file extension matches image regex  
    `\.?(jpg|jpeg|png|gif|bmp|webp|svg|tiff|heic|jfif)$`
  - Left value:  
    `={{ $('Loop Over images').item.binary.data.fileExtension.toLowerCase() }}`
- **Inputs / outputs:**
  - **True branch:** to **Resize Image**
  - **False branch:** back to **Loop Over images** (skip non-image file and continue)
- **Version requirements:** typeVersion **2.2**
- **Potential failures / edge cases:**
  - If `fileExtension` is missing, expression may error (typeValidation is strict). Consider fallback logic.
  - Some valid images may have unusual extensions not included.

#### Node: Resize Image
- **Type / role:** `Edit Image` (preprocessing)
- **Configuration choices:**
  - Operation: **resize**
  - Width: **1000**
  - Height: **1000**
  - `onError: continueRegularOutput` (workflow continues even if resize fails)
- **Inputs / outputs:**
  - Input: binary image from **If is image**
  - Output: resized binary to **Update imageBase64**
- **Version requirements:** typeVersion **1**
- **Potential failures / edge cases:**
  - Unsupported formats (e.g., some HEIC variants) may fail resize.
  - Resizing to fixed 1000x1000 can distort aspect ratio (depending on node behavior/options). Distortion may reduce OCR accuracy.

#### Node: Update imageBase64
- **Type / role:** `Code` (attach Base64 data for API call)
- **Configuration choices:**
  - Gets the **first item** from “Loop Over images” and mutates it (adds `json.imageBase64`).
  - Reads the binary buffer from current input (`Resize Image` output) via:
    - `await this.helpers.getBinaryDataBuffer(0, 'data')`
  - Copies binary from input into `x.binary`.
  - Returns `x` as the new item.
  - `onError: continueRegularOutput` enabled.
- **Key variables/expressions:**
  - `$('Loop Over images').first()`
  - `$input.first().binary`
- **Inputs / outputs:**
  - Input: resized image binary
  - Output: item that includes `json.imageBase64` + binary, to **OCR - openAI**
- **Version requirements:** typeVersion **2**
- **Potential failures / edge cases:**
  - Mutating/returning `x` from another node can cause item-index mismatches in complex loops.
  - If binary property name isn’t `data`, `getBinaryDataBuffer` fails.
  - Base64 increases payload size; may hit OpenAI request limits for large images.

---

### Block C — OpenAI OCR extraction

**Overview:**  
Calls the OpenAI Responses API with both an instruction prompt and the Base64 image, requesting JSON output containing passport fields.

**Nodes involved:**
- OCR - openAI

#### Node: OCR - openAI
- **Type / role:** `HTTP Request` (OpenAI Responses API call)
- **Configuration choices:**
  - Method: **POST**
  - URL: `https://api.openai.com/v1/responses`
  - Body: JSON with:
    - `model: "gpt-4.1-mini"`
    - `input`: user message containing:
      - `input_text`: extraction instructions, including required keys:
        - `type, surname, given_names, nationality, date_of_birth, sex, place_of_birth, date_of_issue, date_of_expiration, cmnd, issuing_authority, issuing_country, overseas_address, passport_no`
      - `input_image`: `image_url` set to `data:image/png;base64,{{ $json.imageBase64 }}`
  - Authentication: `httpBearerAuth` via generic credential
  - `retryOnFail: true`
  - `onError: continueRegularOutput` (continue even if the HTTP request fails)
- **Inputs / outputs:**
  - Input: item with `json.imageBase64`
  - Output: OpenAI Responses API payload (expects `json.output[0].content[0].text` downstream)
  - Connects to **Data standardization**
- **Version requirements:** typeVersion **4.2**
- **Potential failures / edge cases:**
  - Auth misconfiguration (missing/invalid OpenAI API key) → 401/403.
  - Model/endpoint changes (Responses API schema can differ from Chat Completions).
  - If OpenAI returns non-JSON text, downstream JSON parsing may fail (handled with guards).
  - Rate limiting (429), transient network errors (retry helps).
  - Using `data:image/png` regardless of actual format: if source is JPG, still wrapped as PNG in data URL—often tolerated, but not guaranteed. Safer: set proper MIME prefix dynamically.

---

### Block D — Data standardization & validation

**Overview:**  
Parses the model’s response, extracts the first JSON object found, validates key fields, converts dates, generates derived fields (telex transliteration, formatted dates), and sets a passport status.

**Nodes involved:**
- Data standardization
- If has result (disabled)

#### Node: Data standardization
- **Type / role:** `Code` (parsing + normalization)
- **Configuration choices / logic highlights:**
  - Iterates over `$input.all()` (can process multiple OpenAI outputs if batched).
  - Reads: `item.json.output[0].content[0].text`
  - If empty, or no `{...}` match, or missing required keys → pushes `{}` (empty result)
  - Extracts JSON by regex: `/\{[\s\S]*\}/`
  - Validates:
    - `result.type` and `result.given_names` must exist
    - If `issuing_country` is not VN/VNM/VIETNAM/VIET NAM, it still re-checks required fields (redundant but harmless)
  - Date conversions:
    - `convertDate()` supports `dd/mm/yyyy` or Date-parsable strings
    - Creates Date objects for `date_of_birth`, `date_of_issue`, `date_of_expiration`
    - Generates Vietnamese formatted strings `dd/mm/yyyy`
  - Derived fields created (examples):
    - `passport_number` from `passport_no`
    - `issue_date`, `expiry_date`
    - `issuing_place` (telex)
    - `fullname = surname + ' ' + given_names`
    - `fullname_telex`
    - `sex` mapping: `M`→`Male`, `F`→`Female`; also `sex2 = sex.toLowerCase()`
    - `passport_status`:
      - `Expired` if expiration < now
      - `Lost` if expiration > now + 1 year (note: logic is unusual; “Lost” typically isn’t based on expiry date)
    - `cmnd` set to `''` if non-numeric
  - Outputs array of items `{ json: result }` or `{}` placeholders
  - `onError: continueRegularOutput` enabled
- **Inputs / outputs:**
  - Input: OpenAI response JSON from **OCR - openAI**
  - Output: standardized passport JSON items to **If has result**
- **Version requirements:** typeVersion **2**
- **Potential failures / edge cases:**
  - If OpenAI response shape differs (e.g., content array not at `[0]`), `item.json.output[0]...` fails.
  - `JSON.parse` can throw if extracted substring isn’t valid JSON (no try/catch; relies on earlier guards but not foolproof).
  - Date parsing: ambiguous formats (mm/dd vs dd/mm) can create wrong dates if model returns different format.
  - For empty `{}` results, downstream nodes expecting fields may error (but many nodes are disabled/guarded).

#### Node: If has result (disabled)
- **Type / role:** `IF` (intended validation gate)
- **Status:** **Disabled** (node will not execute; n8n bypass behavior depends on version—commonly treated as passthrough or skipped; you should not rely on this gate)
- **Configuration choices:**
  - Condition checks existence of `{{$json.fullname}}`
- **Intended connections:**
  - True → **Prepare QR url**
  - False → **Loop Over images**
- **Version requirements:** typeVersion **2.2**
- **Potential failures / edge cases:**
  - Disabled state means QR generation likely won’t run as designed unless n8n routes around it.
  - If enabled, empty `{}` items would be filtered out (desired behavior).

---

### Block E — QR creation & result aggregation/delivery

**Overview:**  
Transforms standardized passport data into a tab-delimited payload encoded into a QR code URL, aggregates QR results into an HTML message, then outputs to the form completion page and emails it.

**Nodes involved:**
- Prepare QR url
- Grab image
- Form
- Send a message

#### Node: Prepare QR url
- **Type / role:** `Code` (build QR payload and URL)
- **Configuration choices:**
  - Builds a string `qr` with **tab characters** between fields, then URL-encodes it.
  - QR provider: `https://api.qrserver.com/v1/create-qr-code/?size=150x150&data=...`
  - Returns:
    - `url`: QR image URL
    - `name`: `fullname`
- **Key variables/expressions:**
  - `const tab = String.fromCharCode(9)`
  - `encodeURIComponent(qr)`
- **Inputs / outputs:**
  - Input: standardized passport record
  - Output: `{ url, name }` to **Loop Over images** (to continue batching)
- **Version requirements:** typeVersion **2**
- **Potential failures / edge cases:**
  - If required fields are missing, the payload becomes `undefined` in places.
  - External QR service availability, rate limits, or privacy concerns (PII in URL query string).

#### Node: Grab image
- **Type / role:** `Code` (aggregate results into HTML)
- **Configuration choices:**
  - Iterates over `$input.all()`
  - Builds HTML string containing:
    - `[Passport] {name} <br/><img src="{url}"/><br/>`
  - Only includes items where `item.json.url` exists.
  - Returns `{ output }`
- **Inputs / outputs:**
  - Input: stream of items coming from **Loop Over images** (loop output 0)
  - Output: single object to:
    - **Form** (completion)
    - **Send a message** (Gmail)
- **Version requirements:** typeVersion **2**
- **Potential failures / edge cases:**
  - If QR generation never runs (e.g., due to disabled gate), output may be empty.
  - HTML rendering depends on the form completion renderer and email client.

#### Node: Form
- **Type / role:** `Form` (completion page)
- **Configuration choices:**
  - Operation: **completion**
  - Completion title: **Success!**
  - Completion message: `={{$json.output}}` (HTML-like string)
- **Inputs / outputs:**
  - Input: aggregated `{ output }` from **Grab image**
  - Output: ends user interaction (no downstream nodes)
- **Version requirements:** typeVersion **1**
- **Potential failures / edge cases:**
  - Some environments sanitize HTML; images may not render.
  - Large HTML output may be truncated.

#### Node: Send a message
- **Type / role:** `Gmail` (email delivery)
- **Configuration choices:**
  - To: `YOUR_EMAIL` (must be replaced)
  - Subject: `[QR] Passport processing results - {{ $now.format('yyyy-MM-dd hh-mm') }}`
  - Message: `={{ $json.output }}`
  - Credential: Gmail OAuth2 (configured in workflow)
- **Inputs / outputs:**
  - Input: aggregated `{ output }` from **Grab image**
  - Output: email send result (end)
- **Version requirements:** typeVersion **2.1**
- **Potential failures / edge cases:**
  - OAuth token expiry/consent issues.
  - Gmail sending limits or spam filtering.
  - Sending PII over email is risky; ensure access controls.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| On form submission | Form Trigger | Collect uploaded passport images via n8n form | — | Prepare form | ## Overview; Setup; Customization (see note). / ## Form input processing / ⚠️ Privacy: Personal identity data. Limit access and configure email carefully. |
| Prepare form | Code | Split multi-upload binaries into single-file items | On form submission | Loop Over images | ## Form input processing |
| Loop Over images | Split In Batches | Sequentially iterate each uploaded file; central loopback | Prepare form; If is image (false); If has result (false); Prepare QR url | Grab image; If is image | ## Form input processing / ## Image preprocessing |
| If is image | IF | Validate file extension is an image | Loop Over images | Resize Image (true); Loop Over images (false) | ## Image preprocessing |
| Resize Image | Edit Image | Resize images for OCR | If is image (true) | Update imageBase64 | ## Image preprocessing |
| Update imageBase64 | Code | Convert binary image to Base64 and attach to JSON | Resize Image | OCR - openAI | ## Image preprocessing |
| OCR - openAI | HTTP Request | Send image + prompt to OpenAI Responses API for OCR extraction | Update imageBase64 | Data standardization | ## Data extraction and QR generation |
| Data standardization | Code | Parse model output JSON; normalize dates; derive fields; validate | OCR - openAI | If has result | ## Data extraction and QR generation |
| If has result | IF (disabled) | Gate to only generate QR when extraction succeeded | Data standardization | Prepare QR url (true); Loop Over images (false) | ## Data extraction and QR generation |
| Prepare QR url | Code | Create tab-delimited payload and QR-code URL | If has result (true, intended) | Loop Over images | ## Data extraction and QR generation |
| Grab image | Code | Aggregate all QR URLs into one HTML output | Loop Over images | Form; Send a message | ## Result aggregation |
| Form | Form (completion) | Show results on form completion page | Grab image | — | ## Result aggregation |
| Send a message | Gmail | Email aggregated results | Grab image | — | ## Result aggregation |

Sticky note “## Overview … Setup … Customization …” content (preserved contextually):
- Explains the 6-step flow, setup reminders (OpenAI creds, Gmail, replace YOUR_EMAIL), and customization ideas.

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: *Extract passport data with OpenAI and create QR codes* (or your preferred name).

2. **Add “On form submission” (Form Trigger)**
   - Node type: **Form Trigger**
   - Form title: `Process passport images`
   - Description: `Upload multiple passport images and generate QR codes`
   - Add a field:
     - Type: **File**
     - Label: `Passport images`
     - Required: **true**
   - Keep default webhook settings.

3. **Add “Prepare form” (Code)**
   - Node type: **Code**
   - Paste logic to split `items[0].binary` into multiple items (one per uploaded file) and set:
     - `binary.data` per item
     - `json.fileName`, `json.mimeType`
   - Connect: **On form submission → Prepare form**

4. **Add “Loop Over images” (Split In Batches)**
   - Node type: **Split In Batches**
   - In **Options**, set **Reset** expression to:  
     `{{ $prevNode.name == 'On form submission' }}`
   - Connect: **Prepare form → Loop Over images**

5. **Add “If is image” (IF)**
   - Node type: **IF**
   - Condition (String → Regex):
     - Left value: `{{ $('Loop Over images').item.binary.data.fileExtension.toLowerCase() }}`
     - Right value: `\.?(jpg|jpeg|png|gif|bmp|webp|svg|tiff|heic|jfif)$`
   - Connect: **Loop Over images → If is image**
   - Connect **If false → Loop Over images** (to skip non-images)

6. **Add “Resize Image” (Edit Image)**
   - Node type: **Edit Image**
   - Operation: **Resize**
   - Width: `1000`, Height: `1000`
   - Set node error handling to “Continue on Fail” (equivalent to `onError: continueRegularOutput`)
   - Connect: **If is image (true) → Resize Image**

7. **Add “Update imageBase64” (Code)**
   - Node type: **Code**
   - Implement:
     - Read resized binary buffer from property `data`
     - Convert to Base64
     - Store in `json.imageBase64`
     - Ensure the item retains binary for traceability if needed
   - Set “Continue on Fail”
   - Connect: **Resize Image → Update imageBase64**

8. **Add “OCR - openAI” (HTTP Request)**
   - Node type: **HTTP Request**
   - Method: **POST**
   - URL: `https://api.openai.com/v1/responses`
   - Authentication: **Bearer token**
     - Create credentials: **HTTP Bearer Auth**
     - Token: your **OpenAI API key**
   - Body content type: **JSON**
   - JSON body:
     - `model`: `gpt-4.1-mini`
     - `input`: a user message with:
       - text instructions requesting JSON fields
       - image content using `data:image/png;base64,{{ $json.imageBase64 }}`
   - Enable **Retry on Fail**
   - Set “Continue on Fail”
   - Connect: **Update imageBase64 → OCR - openAI**

9. **Add “Data standardization” (Code)**
   - Node type: **Code**
   - Implement parsing/validation:
     - Read `item.json.output[0].content[0].text`
     - Extract JSON object substring
     - `JSON.parse`
     - Validate required fields
     - Normalize dates and create derived fields
     - Output `{ json: result }` per item; output `{}` for invalid extractions
   - Set “Continue on Fail”
   - Connect: **OCR - openAI → Data standardization**

10. **Add “If has result” (IF) — and enable it**
   - Node type: **IF**
   - Condition: “String exists”
     - Left value: `{{ $json.fullname }}`
   - Connect: **Data standardization → If has result**
   - Connect:
     - **True → Prepare QR url**
     - **False → Loop Over images**
   - Note: In the provided workflow this node is disabled; enabling it is required for correct gating.

11. **Add “Prepare QR url” (Code)**
   - Node type: **Code**
   - Build:
     - `qr` = tab-delimited string of specific fields
     - `url` = `https://api.qrserver.com/v1/create-qr-code/?size=150x150&data=${encodeURIComponent(qr)}`
     - `name` = fullname
   - Connect: **Prepare QR url → Loop Over images** (to continue the batch loop)

12. **Add “Grab image” (Code)**
   - Node type: **Code**
   - Aggregate all items so far into a single HTML string with embedded `<img src="...">`.
   - Connect: **Loop Over images → Grab image**
   - (This assumes your loop node produces items that include `json.url` at some point; ensure your batching/merge logic results in that.)

13. **Add “Form” (Form completion)**
   - Node type: **Form**
   - Operation: **Completion**
   - Title: `Success!`
   - Message: `{{ $json.output }}`
   - Connect: **Grab image → Form**

14. **Add “Send a message” (Gmail)**
   - Node type: **Gmail**
   - Operation: **Send**
   - To: replace `YOUR_EMAIL` with your recipient
   - Subject: `[QR] Passport processing results - {{ $now.format('yyyy-MM-dd hh-mm') }}`
   - Message: `{{ $json.output }}`
   - Credentials: configure **Gmail OAuth2** (Google Cloud OAuth consent + scopes as required by n8n)
   - Connect: **Grab image → Send a message**

15. **Add sticky notes (optional but recommended)**
   - Add notes for: Overview, Form input processing, Image preprocessing, Data extraction and QR generation, Result aggregation, Privacy warning.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “This workflow extracts structured passport data using OpenAI OCR and generates QR codes from the extracted information.” | Workflow overview sticky note |
| Setup reminders: add OpenAI API credentials to OCR node; connect Gmail; replace `YOUR_EMAIL`; test with clear passport images | Configuration prerequisites |
| Customization: modify OCR prompt for other documents; adjust QR format | Extensibility |
| ⚠️ Privacy: Personal identity data. Limit access and configure email carefully. | Data protection / operational security |
| QR code provider used: `https://api.qrserver.com/v1/create-qr-code/` | External dependency (availability + PII in URL) |


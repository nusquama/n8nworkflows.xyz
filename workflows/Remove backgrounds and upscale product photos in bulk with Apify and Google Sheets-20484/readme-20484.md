Remove backgrounds and upscale product photos in bulk with Apify and Google Sheets

https://n8nworkflows.xyz/workflows/remove-backgrounds-and-upscale-product-photos-in-bulk-with-apify-and-google-sheets-20484


# Remove backgrounds and upscale product photos in bulk with Apify and Google Sheets

### 1. Workflow Overview

This workflow automates the bulk processing of product images by reading image URLs from a Google Sheet, removing their backgrounds, upscaling the resulting cutouts, and updating the original rows with the processed image data, dimensions, and execution status. 

The primary target use cases are e-commerce catalog management and marketplace asset preparation where large batches of product photos require clean transparent backgrounds and high-resolution scaling.

The logic is grouped into the following functional blocks:
- **1.1 Input Reception & Preparation:** Triggers the workflow, fetches source data from Google Sheets, deduplicates entries, and packages the payload for external processing.
- **1.2 Background Removal Processing:** Interacts with the Apify platform to process image batches and generate transparent PNG cutouts.
- **1.3 Conditional Evaluation & Upscaling:** Evaluates batch outcomes, routes successful cutouts to an image upscaling service, and aggregates successes alongside failure reasons.
- **1.4 Output Persistence:** Maps processed results back to their corresponding sheet row identifiers and writes status updates, URLs, and dimensions back to Google Sheets.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Preparation
- **Overview:** Initializes the execution, extracts product image records from a designated spreadsheet, and structures the data into a deduplicated batch format to optimize external API calls.
- **Nodes Involved:** 
  - `Start`
  - `Read image URLs`
  - `Collect image URLs`
- **Node Details:**
  - **Start**
    - *Type and technical role:* `n8n-nodes-base.manualTrigger` — Acts as the manual entry point for the workflow.
    - *Configuration choices:* Default manual execution configuration.
    - *Input/Output connections:* Input: None; Output: `Read image URLs`.
    - *Edge cases:* None.
  - **Read image URLs**
    - *Type and technical role:* `n8n-nodes-base.googleSheets` — Reads rows from a connected Google Sheets document.
    - *Configuration choices:* Operation set to `read`. Spreadsheet and sheet name must be configured dynamically or selected from available integrations.
    - *Input/Output connections:* Input: `Start`; Output: `Collect image URLs`.
    - *Credentials:* Requires Google Sheets OAuth2 authentication.
    - *Edge cases:* Authentication token expiration, missing spreadsheet permissions, or empty data sets causing downstream parsing anomalies.
  - **Collect image URLs**
    - *Type and technical role:* `n8n-nodes-base.code` — JavaScript-based data transformation node that extracts, trims, and deduplicates image URLs while maintaining row-to-index mapping.
    - *Key expressions or variables used:* Iterates through `$input.all()`, tracking `item.json.image_url` and mapping row numbers to unique array indices.
    - *Input/Output connections:* Input: `Read image URLs`; Output: `Remove backgrounds`.
    - *Edge cases:* Empty sheets return an empty array early to prevent unnecessary API charges.

#### 2.2 Background Removal Processing
- **Overview:** Sends the batch of deduplicated image URLs to an Apify actor that processes the images and returns transparent background cutouts.
- **Nodes Involved:** 
  - `Remove backgrounds`
- **Node Details:**
  - **Remove backgrounds**
    - *Type and technical role:* `@apify/n8n-nodes-apify.apify` — Executes an Apify actor and waits for dataset results.
    - *Configuration choices:* Resource set to `Actors`, operation set to `Run actor and get dataset`. Actor ID: `sherwood~background-remover-batch`. Memory allocated: `1024` MB. Maximum total charge limit: `$5`. Custom body passes `{ imageUrls: $json.imageUrls, mode: 'general', outputFormat: 'png' }`.
    - *Input/Output connections:* Input: `Collect image URLs`; Output: `Prepare upscaling`.
    - *Credentials:* Requires Apify API token credentials.
    - *Edge cases:* Exceeding the `$5` max charge limit, actor timeouts, or invalid image URLs returned as HTTP errors from the source.

#### 2.3 Conditional Evaluation & Upscaling
- **Overview:** Analyzes the background removal output, separates successful cutouts from errors, verifies if any valid images remain, and routes them to an upscaling actor.
- **Nodes Involved:** 
  - `Prepare upscaling`
  - `Any cutouts?`
  - `Upscale cutouts`
- **Node Details:**
  - **Prepare upscaling**
    - *Type and technical role:* `n8n-nodes-base.code` — JavaScript node that sorts background removal results by index, associates cutout URLs with all matching source row numbers, and flags failed operations.
    - *Key expressions or variables used:* References data from `$('Collect image URLs').first().json` and current node inputs.
    - *Input/Output connections:* Input: `Remove backgrounds`; Output: `Any cutouts?`.
    - *Edge cases:* All images failing background removal results in an empty `cutoutUrls` array.
  - **Any cutouts?**
    - *Type and technical role:* `n8n-nodes-base.if` — Branching node that determines whether upscaling is necessary based on the presence of successful cutouts.
    - *Key expressions or variables used:* Evaluates `{{ $json.cutoutUrls.length }}` > `0`.
    - *Input/Output connections:* Input: `Prepare upscaling`; Outputs: True branch connects to `Upscale cutouts`, False branch routes directly to `Match results to rows`.
    - *Edge cases:* Zero successful cutouts bypasses the upscaling step entirely to save compute time and cost.
  - **Upscale cutouts**
    - *Type and technical role:* `@apify/n8n-nodes-apify.apify` — Executes the image upscaling Apify actor on the verified cutout batch.
    - *Configuration choices:* Resource set to `Actors`, operation set to `Run actor and get dataset`. Actor ID: `sherwood~image-upscaler-batch`. Memory allocated: `1024` MB. Maximum total charge limit: `$5`. Custom body passes `{ imageUrls: $json.cutoutUrls, scale: '2', outputFormat: 'png' }`.
    - *Input/Output connections:* Input: `Any cutouts?` (True branch); Output: `Match results to rows`.
    - *Credentials:* Requires Apify API token credentials.
    - *Edge cases:* Upscaling actor failures, malformed cutout response structures, or API rate limiting.

#### 2.4 Output Persistence
- **Overview:** Consolidates successful upscaled image URLs, dimensions, and error logs, matches them back to their initial spreadsheet row numbers, and updates the target sheet in bulk.
- **Nodes Involved:** 
  - `Match results to rows`
  - `Write results`
- **Node Details:**
  - **Match results to rows**
    - *Type and technical role:* `n8n-nodes-base.code` — JavaScript data mapping node that merges upscaled results and background removal error logs into a uniform schema aligned with spreadsheet row indices.
    - *Key expressions or variables used:* References `$('Prepare upscaling').first().json` and input items to compile final object arrays containing `row_number`, `result_url`, `width`, `height`, `status`, and `error`.
    - *Input/Output connections:* Inputs: `Upscale cutouts` and `Any cutouts?` (False branch); Output: `Write results`.
    - *Edge cases:* Mismatched array lengths or missing indices if payload sequences are altered.
  - **Write results**
    - *Type and technical role:* `n8n-nodes-base.googleSheets` — Writes processed results back to the source spreadsheet.
    - *Configuration choices:* Operation set to `update`. Mapping mode set to `autoMapInputData`. Matching column configured to `row_number`.
    - *Input/Output connections:* Input: `Match results to rows`; Output: None.
    - *Credentials:* Requires Google Sheets OAuth2 authentication.
    - *Edge cases:* Row locking conflicts, exceeding Google Sheets API rate limits during bulk writes, or mismatching row identification keys.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Overview | n8n-nodes-base.stickyNote | Documentation block outlining workflow purpose, architecture, and configuration prerequisites. | None | None | ## Remove backgrounds and upscale product photos in bulk<br><br>**Who's it for:** e-commerce teams and freelancers who receive product photos with busy backgrounds or low resolution and need clean, large images for a shop or a marketplace.<br><br>**How it works**<br>1. Reads image URLs from a Google Sheet.<br>2. Sends them in one batch to the **Background Remover API** actor on Apify, which returns a transparent PNG per image.<br>3. Sends the cutouts to the **Image Upscaler API** actor, which upscales each one 2x (change `scale` to `4` in the node).<br>4. Writes the final image URL, width and height back to the sheet, next to the original. Failed images are marked with the reason and are not charged.<br><br>**How to set up**<br>- Create a sheet with the columns `image_url`, `result_url`, `width`, `height`, `status`, `error` and fill `image_url`.<br>- Connect Google Sheets in the first and last nodes and pick your sheet.<br>- Add your Apify API token as an Apify credential in both Apify nodes (install the verified Apify node if needed).<br><br>**Requirements:** an Apify account (pay per image, no subscription to the actors) and a Google account.<br><br>**Customize:** replace Google Sheets with Airtable, Google Drive or a webhook; remove the upscaling step if you only need cutouts. Each Apify node has a maximum cost per run of $5 that you can raise. |
| Sheet columns | n8n-nodes-base.stickyNote | Reference note defining expected spreadsheet column schema requirements. | None | None | ### Sheet columns<br>`image_url` (input), then `result_url`, `width`, `height`, `status`, `error` (filled by the workflow). |
| Start | n8n-nodes-base.manualTrigger | Manual workflow execution trigger. | None | Read image URLs | |
| Read image URLs | n8n-nodes-base.googleSheets | Reads raw source rows containing product image URLs from Google Sheets. | Start | Collect image URLs | |
| Collect image URLs | n8n-nodes-base.code | Deduplicates image URLs and maps indices to source rows. | Read image URLs | Remove backgrounds | |
| Remove backgrounds | @apify/n8n-nodes-apify.apify | Calls Apify Background Remover actor to strip image backgrounds in batch. | Collect image URLs | Prepare upscaling | |
| Prepare upscaling | n8n-nodes-base.code | Sorts background removal outputs and segments successful cutouts from failures. | Remove backgrounds | Any cutouts? | |
| Any cutouts? | n8n-nodes-base.if | Conditional gateway verifying if any successful cutouts are available for upscaling. | Prepare upscaling | Upscale cutouts, Match results to rows | |
| Upscale cutouts | @apify/n8n-nodes-apify.apify | Calls Apify Image Upscaler actor to enlarge cutout images 2x. | Any cutouts? | Match results to rows | |
| Match results to rows | n8n-nodes-base.code | Consolidates successful upscaled outputs and failure logs into row-mapped objects. | Upscale cutouts, Any cutouts? | Write results | |
| Write results | n8n-nodes-base.googleSheets | Updates Google Sheets records with processing status, URLs, and dimensions. | Match results to rows | None | |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Entry Node:** Add a **Manual Trigger** (`n8n-nodes-base.manualTrigger`) node named `Start`.
2. **Add Data Ingestion:** Create a **Google Sheets** (`n8n-nodes-base.googleSheets`) node named `Read image URLs`. Connect `Start` to it. Set operation to `read`, and configure your target document and sheet.
3. **Configure Deduplication Logic:** Add a **Code** (`n8n-nodes-base.code`) node named `Collect image URLs`. Connect `Read image URLs` to it. Populate the JavaScript execution block to filter out empty strings, deduplicate URLs, and track row indices.
4. **Add Background Removal Integration:** Create an **Apify** (`@apify/n8n-nodes-apify.apify`) node named `Remove backgrounds`. Connect `Collect image URLs` to it. Configure Resource as `Actors`, Operation as `Run actor and get dataset`, Actor ID as `sherwood~background-remover-batch`, Memory as `1024`, Max Total Charge as `5`, and supply the custom JSON body for batch processing. Attach valid Apify API credentials.
5. **Add Preparation Logic:** Create a **Code** (`n8n-nodes-base.code`) node named `Prepare upscaling`. Connect `Remove backgrounds` to it. Add the sorting and categorization script to map results back to row numbers.
6. **Add Conditional Branching:** Create an **If** (`n8n-nodes-base.if`) node named `Any cutouts?`. Connect `Prepare upscaling` to it. Configure a numeric condition checking if `{{ $json.cutoutUrls.length }}` is greater than `0`.
7. **Add Upscaling Integration:** Create an **Apify** (`@apify/n8n-nodes-apify.apify`) node named `Upscale cutouts`. Connect the **True** output of `Any cutouts?` to it. Configure Resource as `Actors`, Operation as `Run actor and get dataset`, Actor ID as `sherwood~image-upscaler-batch`, Memory as `1024`, Max Total Charge as `5`, and supply the custom JSON body with a scale of `2`. Attach Apify API credentials.
8. **Add Results Matching Logic:** Create a **Code** (`n8n-nodes-base.code`) node named `Match results to rows`. Connect both the `Upscale cutouts` node and the **False** output of `Any cutouts?` to this node. Populate the mapping script to build the final update payload.
9. **Configure Data Persistence:** Create a **Google Sheets** (`n8n-nodes-base.googleSheets`) node named `Write results`. Connect `Match results to rows` to it. Set operation to `update`, mapping mode to `autoMapInputData`, and matching column to `row_number`. Select the same target document and sheet, and authenticate using Google Sheets credentials.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Apify Background Remover Actor Documentation | [Apify Background Remover Actor](https://apify.com/sherwood/background-remover-batch) |
| Apify Image Upscaler Actor Documentation | [Apify Image Upscaler Actor](https://apify.com/sherwood/image-upscaler-batch) |
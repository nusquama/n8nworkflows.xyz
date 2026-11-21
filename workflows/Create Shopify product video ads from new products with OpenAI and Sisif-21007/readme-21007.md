Create Shopify product video ads from new products with OpenAI and Sisif

https://n8nworkflows.xyz/workflows/create-shopify-product-video-ads-from-new-products-with-openai-and-sisif-21007


# Create Shopify product video ads from new products with OpenAI and Sisif

### 1. Workflow Overview

This workflow automates the creation and publishing of short, AI-generated product video advertisements for new Shopify store items. Whenever a merchant creates a product in Shopify, the workflow extracts its details, uses OpenAI to write an optimized motion prompt, and submits the primary product image to Sisif for image-to-video rendering. It polls the rendering status until completion, uploads the resulting MP4 directly to Shopify via the Admin GraphQL API as product media, falls back to Google Drive if any upload step fails, and reports the final status to a designated Slack channel.

The workflow is structured into the following logical functional blocks:
- **1.1 Input Reception & Configuration:** Captures newly created products from Shopify, loads global workspace variables, and sanitizes/extracts essential product attributes.
- **1.2 AI Prompt Generation & Video Initiation:** Validates the presence of a valid product image, leverages an LLM to generate a cinematic motion prompt, and initiates the video generation job with Sisif.
- **1.3 Video Generation Polling Loop:** Periodically checks the status of the video render job, handling successful completion, errors, refunds, or timeouts.
- **1.4 Shopify Upload & Attachment Pipeline:** Downloads the rendered video, measures its size, creates a secure Shopify staged upload target, performs a multipart upload, and attaches the video asset to the Shopify product.
- **1.5 Fallback & Notification System:** Handles errors by falling back to Google Drive storage and dispatches status updates to Slack.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Input Reception & Configuration
- **Overview:** This block intercepts new product creation events from Shopify, initializes necessary environment variables, and extracts core metadata required for content generation.
- **Nodes Involved:** 
  - `Shopify Trigger`
  - `Workflow Config`
  - `Extract Product Data`
  - `Has Main Image?`
- **Node Details:**
  - **Shopify Trigger**
    - *Type & Role:* `n8n-nodes-base.shopifyTrigger` (Trigger Node) - Listens for webhook events when a product is created.
    - *Configuration:* Topic set to `products/create` using Shopify Access Token authentication.
    - *Key Expressions:* None.
    - *Connections:* Input: None | Output: `Workflow Config`.
    - *Edge Cases:* Webhook delivery failures, invalid Shopify credentials, or network timeouts during registration.
  - **Workflow Config**
    - *Type & Role:* `n8n-nodes-base.set` (Data Transformation) - Establishes static operational parameters for the run.
    - *Configuration:* Defines global variables: `shop_subdomain`, `shopify_api_version` (default `2026-07`), `drive_folder_id`, and `slack_channel`.
    - *Key Expressions:* Static string assignments.
    - *Connections:* Input: `Shopify Trigger` | Output: `Extract Product Data`.
    - *Edge Cases:* Missing configuration values causing downstream API routing failures.
  - **Extract Product Data**
    - *Type & Role:* `n8n-nodes-base.set` (Data Transformation) - Parses raw webhook payload into standardized fields.
    - *Configuration:* Maps JSON properties to custom fields: `product_gid`, `title`, HTML-stripped `description` (truncated to 800 characters), `product_type`, and `image_url`.
    - *Key Expressions:* Uses JavaScript string manipulation and regex tag stripping (`.replace(/<[^>]*>/g, ' ')`).
    - *Connections:* Input: `Workflow Config` | Output: `Has Main Image?`.
    - *Edge Cases:* Missing fields in the webhook payload resulting in empty variables.
  - **Has Main Image?**
    - *Type & Role:* `n8n-nodes-base.if` (Flow Control) - Filters out products lacking a valid image URL.
    - *Configuration:* Evaluates if `image_url` starts with `https://`.
    - *Key Expressions:* `={{ $json.image_url }}` evaluated against `https://`.
    - *Connections:* Input: `Extract Product Data` | Output: `Write Motion Prompt` (True branch).
    - *Edge Cases:* Products without images are silently dropped (false branch unconnected).

---

#### Block 1.2: AI Prompt Generation & Video Initiation
- **Overview:** Translates product attributes into a tailored cinematic motion prompt using an LLM and submits an asynchronous generation request to the Sisif API.
- **Nodes Involved:**
  - `Write Motion Prompt`
  - `OpenAI Chat Model`
  - `Generate Video from Image`
- **Node Details:**
  - **Write Motion Prompt**
    - *Type & Role:* `@n8n/n8n-nodes-langchain.chainLlm` (AI Agent / Chain) - Generates a concise camera motion prompt based on product details.
    - *Configuration:* Uses a system/user prompt template constrained to under 60 words, focusing on subtle camera work (slow push-in, orbit, parallax).
    - *Key Expressions:* Injects `{{ $json.title }}`, `{{ $json.product_type }}`, and `{{ $json.description }}`.
    - *Connections:* Input: `Has Main Image?` | Output: `Generate Video from Image`.
    - *Edge Cases:* LLM token limits or API quota exhaustion.
  - **OpenAI Chat Model**
    - *Type & Role:* `@n8n/n8n-nodes-langchain.lmChatOpenAi` (AI Model Provider) - Supplies the underlying chat model (`gpt-4o-mini`) for the prompt generation chain.
    - *Configuration:* Configured with OpenAI API credentials.
    - *Key Expressions:* None.
    - *Connections:* Input: None (linked via AI model binding to `Write Motion Prompt`) | Output: `Write Motion Prompt`.
    - *Edge Cases:* Invalid OpenAI API keys or rate-limiting errors.
  - **Generate Video from Image**
    - *Type & Role:* `n8n-nodes-base.httpRequest` (API Request) - Submits the image URL and generated text prompt to Sisif.
    - *Configuration:* POST request to `https://sisif.ai/api/videos/generate/` using HTTP Bearer Token authentication.
    - *Key Expressions:* `={{ $json.text.trim() }}` for prompt, `={{ $('Extract Product Data').item.json.image_url }}` for image.
    - *Connections:* Input: `Write Motion Prompt` | Output: `Wait 30s`.
    - *Edge Cases:* Invalid API bearer token, malformed image URLs rejected by Sisif, or insufficient credit balance on the Sisif account.

---

#### Block 1.3: Video Generation Polling Loop
- **Overview:** Implements a polling loop that queries the Sisif rendering status every 30 seconds until the video is ready, fails, or reaches the maximum check limit.
- **Nodes Involved:**
  - `Wait 30s`
  - `Check Video Status`
  - `Video Ready?`
  - `Render Failed?`
  - `Max Attempts Reached?`
  - `Mark Timed Out`
  - `Notify Slack: Render Failed`
- **Node Details:**
  - **Wait 30s**
    - *Type & Role:* `n8n-nodes-base.wait` (Flow Control / Delay) - Pauses execution for 30 seconds between status checks.
    - *Configuration:* Unit set to seconds, amount set to 30.
    - *Connections:* Input: `Generate Video from Image` or `Max Attempts Reached?` (loop back) | Output: `Check Video Status`.
  - **Check Video Status**
    - *Type & Role:* `n8n-nodes-base.httpRequest` (API Request) - Queries the current status of the render job.
    - *Configuration:* GET request to `https://sisif.ai/api/videos/{{ $json.id }}/status/` using HTTP Bearer Authentication.
    - *Connections:* Input: `Wait 30s` | Output: `Video Ready?`.
    - *Edge Cases:* Intermittent network dropouts or API rate limits.
  - **Video Ready?**
    - *Type & Role:* `n8n-nodes-base.if` (Flow Control) - Evaluates if the render status is `READY`.
    - *Configuration:* Checks if `status` equals `READY`.
    - *Connections:* Input: `Check Video Status` | Output: `Download Video` (True) or `Render Failed?` (False).
  - **Render Failed?**
    - *Type & Role:* `n8n-nodes-base.if` (Flow Control) - Checks if the render resulted in an explicit terminal failure.
    - *Configuration:* Regex match `^(ERROR|REFUNDED)$` against `status`.
    - *Connections:* Input: `Video Ready?` (False branch) | Output: `Notify Slack: Render Failed` (True) or `Max Attempts Reached?` (False).
  - **Max Attempts Reached?**
    - *Type & Role:* `n8n-nodes-base.if` (Flow Control) - Counts execution iterations to prevent infinite polling loops.
    - *Configuration:* Checks if `$runIndex + 1 >= 40` (approx. 20 minutes total timeout).
    - *Connections:* Input: `Render Failed?` (False branch) | Output: `Mark Timed Out` (True) or `Wait 30s` (False branch, looping back).
  - **Mark Timed Out**
    - *Type & Role:* `n8n-nodes-base.set` (Data Transformation) - Formats timeout error metadata.
    - *Configuration:* Sets `status` to `TIMEOUT` and provides a descriptive `error_message`.
    - *Connections:* Input: `Max Attempts Reached?` (True branch) | Output: `Notify Slack: Render Failed`.
  - **Notify Slack: Render Failed**
    - *Type & Role:* `n8n-nodes-base.slack` (Messaging Integration) - Alerts the operations team of render failure or timeout.
    - *Configuration:* Posts a notification message to the configured Slack channel.
    - *Connections:* Input: `Render Failed?` or `Mark Timed Out` | Output: None (Terminal node).

---

#### Block 1.4: Shopify Upload & Attachment Pipeline
- **Overview:** Downloads the completed MP4 video, calculates its byte size, requests an authenticated staged upload target from Shopify, uploads the file via multipart form-data, and associates the media with the product.
- **Nodes Involved:**
  - `Download Video`
  - `Measure Video File`
  - `Create Staged Upload`
  - `Download Video for Upload`
  - `Prepare Staged Upload`
  - `Staged Target Ready?`
  - `Upload to Staged Target`
  - `Upload Succeeded?`
  - `Attach Video to Product`
  - `Attached to Shopify?`
- **Node Details:**
  - **Download Video**
    - *Type & Role:* `n8n-nodes-base.httpRequest` (File Download) - Downloads the rendered MP4 file from Sisif as binary data.
    - *Configuration:* Response format set to file.
    - *Connections:* Input: `Video Ready?` (True) | Output: `Measure Video File`.
    - *Edge Cases:* Large file downloads timing out or broken source URLs.
  - **Measure Video File**
    - *Type & Role:* `n8n-nodes-base.code` (JavaScript execution) - Determines the exact byte length of the binary buffer required by Shopify's GraphQL API.
    - *Configuration:* Iterates binary items, reads buffer length, and normalizes a sanitized filename (`<title>-sisif.mp4`).
    - *Connections:* Input: `Download Video` | Output: `Create Staged Upload`.
  - **Create Staged Upload**
    - *Type & Role:* `n8n-nodes-base.httpRequest` (GraphQL API Mutation) - Requests a secure upload target URL and required parameters from Shopify.
    - *Configuration:* POST request to Shopify GraphQL endpoint using Shopify Access Token credentials, executing the `stagedUploadsCreate` mutation for resource type `VIDEO`.
    - *Connections:* Input: `Measure Video File` | Output: `Download Video for Upload`.
    - *Edge Cases:* API schema changes, scope deficiencies (`write_files` missing), or invalid file sizes.
  - **Download Video for Upload**
    - *Type & Role:* `n8n-nodes-base.httpRequest` (File Download) - Re-downloads the source video binary to ensure it is fresh and correctly attached to the execution payload.
    - *Configuration:* Response format set to file.
    - *Connections:* Input: `Create Staged Upload` | Output: `Prepare Staged Upload`.
  - **Prepare Staged Upload**
    - *Type & Role:* `n8n-nodes-base.code` (Data Validation & Transformation) - Validates Shopify's returned staged upload target and parameters.
    - *Configuration:* Checks for user errors and verifies that parameter counts do not exceed expected form constraints.
    - *Connections:* Input: `Download Video for Upload` | Output: `Staged Target Ready?`.
  - **Staged Target Ready?**
    - *Type & Role:* `n8n-nodes-base.if` (Flow Control) - Validates whether the upload target preparation succeeded.
    - *Configuration:* Evaluates `upload_ready === true`.
    - *Connections:* Input: `Prepare Staged Upload` | Output: `Upload to Staged Target` (True) or `Download Video for Drive` (False fallback).
  - **Upload to Staged Target**
    - *Type & Role:* `n8n-nodes-base.httpRequest` (Multipart File Upload) - Uploads the MP4 file to Google Cloud Storage via Shopify's generated staged parameters.
    - *Configuration:* POST request to the dynamic `upload_url` with `multipart-form-data`, passing up to 12 indexed parameter slots followed by the binary file field (`file`).
    - *Connections:* Input: `Staged Target Ready?` (True) | Output: `Upload Succeeded?`.
    - *Edge Cases:* Network interruptions or HTTP 4xx/5xx responses from the storage bucket.
  - **Upload Succeeded?**
    - *Type & Role:* `n8n-nodes-base.if` (Flow Control) - Verifies successful HTTP status code from the staged target upload.
    - *Configuration:* Checks for `!json.error` and HTTP status codes in the `200–299` range.
    - *Connections:* Input: `Upload to Staged Target` | Output: `Attach Video to Product` (True) or `Download Video for Drive` (False fallback).
  - **Attach Video to Product**
    - *Type & Role:* `n8n-nodes-base.httpRequest` (GraphQL API Mutation) - Links the uploaded staged media resource to the Shopify product via `productUpdate`.
    - *Configuration:* POST request to Shopify GraphQL endpoint using Access Token authentication.
    - *Connections:* Input: `Upload Succeeded?` (True) | Output: `Attached to Shopify?`.
    - *Edge Cases:* GraphQL user errors returned in the response payload.
  - **Attached to Shopify?**
    - *Type & Role:* `n8n-nodes-base.if` (Flow Control) - Validates whether the GraphQL mutation executed without errors or user errors.
    - *Configuration:* Evaluates absence of errors and empty `productUpdate.userErrors` array.
    - *Connections:* Input: `Attach Video to Product` | Output: `Notify Slack: Attached` (True) or `Download Video for Drive` (False fallback).

---

#### Block 1.5: Fallback & Notification System
- **Overview:** Handles successful delivery notifications to Slack or invokes a Google Drive fallback upload if any Shopify delivery step fails.
- **Nodes Involved:**
  - `Notify Slack: Attached`
  - `Download Video for Drive`
  - `Upload to Google Drive`
  - `Notify Slack: Saved to Drive`
- **Node Details:**
  - **Notify Slack: Attached**
    - *Type & Role:* `n8n-nodes-base.slack` (Messaging Integration) - Sends a success message to Slack confirming video attachment to Shopify.
    - *Configuration:* Uses Slack credentials and routes messages to the configured channel.
    - *Connections:* Input: `Attached to Shopify?` (True) | Output: None (Terminal node).
  - **Download Video for Drive**
    - *Type & Role:* `n8n-nodes-base.httpRequest` (File Download) - Downloads the video for fallback storage when Shopify integration fails.
    - *Configuration:* Response format set to file from `Check Video Status` video URL.
    - *Connections:* Input: `Staged Target Ready?` (False), `Upload Succeeded?` (False), or `Attached to Shopify?` (False) | Output: `Upload to Google Drive`.
  - **Upload to Google Drive**
    - *Type & Role:* `n8n-nodes-base.googleDrive` (Storage Integration) - Uploads the MP4 file to a designated Google Drive folder.
    - *Configuration:* Uses Google Drive OAuth2 credentials, sets file name to `<title> - sisif.mp4`, and targets the folder ID from `Workflow Config`.
    - *Connections:* Input: `Download Video for Drive` | Output: `Notify Slack: Saved to Drive`.
    - *Edge Cases:* Invalid Google Drive folder IDs or expired OAuth2 tokens.
  - **Notify Slack: Saved to Drive**
    - *Type & Role:* `n8n-nodes-base.slack` (Messaging Integration) - Notifies the team that the video was saved to Google Drive due to a Shopify publishing fallback.
    - *Configuration:* Posts message containing the Google Drive web view link and Sisif source URL.
    - *Connections:* Input: `Upload to Google Drive` | Output: None (Terminal node).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Setup Instructions** | `n8n-nodes-base.stickyNote` | Documentation note | None | None | Turn new Shopify products into AI product video ads with Sisif... [Setup & Customization details] |
| **Note: Generate and Poll** | `n8n-nodes-base.stickyNote` | Documentation note | None | None | ### Sisif render loop<br>The photo URL is sent as `image` with the motion `prompt`... |
| **Note: Attach or Fallback** | `n8n-nodes-base.stickyNote` | Documentation note | None | None | ### Deliver the video to Shopify<br>Shopify does not accept an external video URL... |
| **Shopify Trigger** | `n8n-nodes-base.shopifyTrigger` | Webhook entry point | None | Workflow Config | 1. **Shopify Trigger** fires when a product is created. |
| **Workflow Config** | `n8n-nodes-base.set` | Variable initialization | Shopify Trigger | Extract Product Data | 3. Open **Workflow Config** and set `shop_subdomain`... |
| **Extract Product Data** | `n8n-nodes-base.set` | Data transformation | Workflow Config | Has Main Image? | 2. The main product image and copy are extracted. Products without an image are skipped. |
| **Has Main Image?** | `n8n-nodes-base.if` | Conditional branching | Extract Product Data | Write Motion Prompt | 2. The main product image and copy are extracted. Products without an image are skipped. |
| **Write Motion Prompt** | `@n8n/n8n-nodes-langchain.chainLlm` | AI Prompt Generator | Has Main Image? | Generate Video from Image | 3. An AI model writes a short camera/motion prompt for that product. |
| **OpenAI Chat Model** | `@n8n/n8n-nodes-langchain.lmChatOpenAi` | LLM Provider | None (AI Binding) | Write Motion Prompt | 4. Add OpenAI (or swap in any chat model)... |
| **Generate Video from Image** | `n8n-nodes-base.httpRequest` | API Video Request | Write Motion Prompt | Wait 30s | 4. **Sisif** animates the product photo into a 5-second portrait MP4 (image-to-video).<br>### Sisif render loop<br>The photo URL is sent as `image` with the motion `prompt`... |
| **Wait 30s** | `n8n-nodes-base.wait` | Execution delay | Generate Video from Image / Max Attempts Reached? | Check Video Status | 5. The workflow polls every 30 s until the video is `READY`... |
| **Check Video Status** | `n8n-nodes-base.httpRequest` | Status API Request | Wait 30s | Video Ready? | 5. The workflow polls every 30 s until the video is `READY`... |
| **Video Ready?** | `n8n-nodes-base.if` | Polling evaluation | Check Video Status | Download Video / Render Failed? | 5. The workflow polls every 30 s until the video is `READY`... |
| **Render Failed?** | `n8n-nodes-base.if` | Error evaluation | Video Ready? | Notify Slack: Render Failed / Max Attempts Reached? | 5. ...stops on `ERROR`/`REFUNDED`, and times out after 40 checks. |
| **Attached to Shopify?** | `n8n-nodes-base.if` | Success evaluation | Attach Video to Product | Notify Slack: Attached / Download Video for Drive | 6. The MP4 is uploaded to Shopify with a staged upload and added to the product as media... If any Shopify step fails, the MP4 goes to Google Drive and Slack gets the link. |
| **Notify Slack: Attached** | `n8n-nodes-base.slack` | Success notification | Attached to Shopify? | None | 7. A Slack message reports the result. |
| **Upload to Google Drive** | `n8n-nodes-base.googleDrive` | Fallback storage upload | Download Video for Drive | Notify Slack: Saved to Drive | 6. ...If any Shopify step fails, the MP4 is saved to Google Drive instead. |
| **Notify Slack: Saved to Drive** | `n8n-nodes-base.slack` | Fallback notification | Upload to Google Drive | None | 7. A Slack message reports the result. |
| **Notify Slack: Render Failed** | `n8n-nodes-base.slack` | Error notification | Mark Timed Out / Render Failed? | None | 7. A Slack message reports the result. |
| **Max Attempts Reached?** | `n8n-nodes-base.if` | Timeout counter check | Render Failed? | Mark Timed Out / Wait 30s | 5. ...times out after 40 checks (about 20 minutes). |
| **Mark Timed Out** | `n8n-nodes-base.set` | Timeout metadata formatter | Max Attempts Reached? | Notify Slack: Render Failed | 5. ...times out after 40 checks (about 20 minutes). |
| **Download Video** | `n8n-nodes-base.httpRequest` | Video file download | Video Ready? | Measure Video File | 6. The MP4 is uploaded to Shopify with a staged upload... |
| **Measure Video File** | `n8n-nodes-base.code` | Binary size calculator | Download Video | Create Staged Upload | 6. The MP4 is uploaded to Shopify with a staged upload... |
| **Create Staged Upload** | `n8n-nodes-base.httpRequest` | Shopify API GraphQL call | Measure Video File | Download Video for Upload | 6. ...`stagedUploadsCreate` (resource `VIDEO`, method `POST`) returns an upload target... |
| **Upload Succeeded?** | `n8n-nodes-base.if` | HTTP status check | Upload to Staged Target | Attach Video to Product / Download Video for Drive | 6. ...**Upload to Staged Target** posts a multipart form with Shopify's parameters... If any step fails, the MP4 goes to Google Drive... |
| **Attach Video to Product** | `n8n-nodes-base.httpRequest` | Shopify API GraphQL mutation | Upload Succeeded? | Attached to Shopify? | 6. ...and `productUpdate` then adds the video to the product using the returned `resourceUrl`... |
| **Download Video for Drive** | `n8n-nodes-base.httpRequest` | Fallback file download | Staged Target Ready? / Upload Succeeded? / Attached to Shopify? | Upload to Google Drive | 6. If any Shopify step fails, the MP4 is saved to Google Drive instead. |
| **Download Video for Upload** | `n8n-nodes-base.httpRequest` | Binary refresh download | Create Staged Upload | Prepare Staged Upload | 6. The MP4 is uploaded to Shopify with a staged upload... |
| **Prepare Staged Upload** | `n8n-nodes-base.code` | Target validation script | Download Video for Upload | Staged Target Ready? | 6. ...**Upload to Staged Target** posts a multipart form with Shopify's parameters in their original order and the file last. |
| **Staged Target Ready?** | `n8n-nodes-base.if` | Validation gate | Prepare Staged Upload | Upload to Staged Target / Download Video for Drive | 6. ...**Upload to Staged Target** posts a multipart form with Shopify's parameters... If any step fails, the MP4 goes to Google Drive... |
| **Upload to Staged Target** | `n8n-nodes-base.httpRequest` | Multipart upload call | Staged Target Ready? | Upload Succeeded? | 6. ...**Upload to Staged Target** posts a multipart form with Shopify's parameters in their original order and the file last. Up to 12 parameters are supported; unused slots are sent as `x-ignore-*` fields, which Google Cloud Storage ignores. |

---

### 4. Reproducing the Workflow from Scratch

To rebuild this workflow manually in n8n, follow these sequential steps:

1. **Create the Trigger Node:**
   - Add a **Shopify Trigger** node (`n8n-nodes-base.shopifyTrigger`).
   - Configure the topic to `products/create` and set authentication to use a **Shopify Access Token**.

2. **Configure Global Variables:**
   - Add a **Set** node named `Workflow Config`.
   - Add string assignments: `shop_subdomain` (`your-store`), `shopify_api_version` (`2026-07`), `drive_folder_id` (target Google Drive folder ID), and `slack_channel` (`#product-videos`).
   - Connect `Shopify Trigger` to `Workflow Config`.

3. **Extract Product Metadata:**
   - Add a **Set** node named `Extract Product Data`.
   - Map fields: `product_gid` (`={{ $('Shopify Trigger').item.json.admin_graphql_api_id }}`), `title` (`={{ $('Shopify Trigger').item.json.title }}`), `description` (`={{ ($('Shopify Trigger').item.json.body_html || '').replace(/<[^>]*>/g, ' ').slice(0, 800) }}`), `product_type` (`={{ $('Shopify Trigger').item.json.product_type }}`), and `image_url` (`={{ $('Shopify Trigger').item.json.image?.src || $('Shopify Trigger').item.json.images?.[0]?.src || '' }}`).
   - Connect `Workflow Config` to `Extract Product Data`.

4. **Filter Products Without Images:**
   - Add an **If** node named `Has Main Image?`.
   - Set condition: String `={{ $json.image_url }}` starts with `https://`.
   - Connect `Extract Product Data` to `Has Main Image?`.

5. **Set Up AI Prompt Generation:**
   - Add an **Advanced AI Chain (Basic LLM Chain)** node named `Write Motion Prompt`.
   - Configure the prompt template to request a short, under-60-word cinematic motion prompt incorporating product title, type, and description.
   - Add an **OpenAI Chat Model** node (`gpt-4o-mini`) and link it to the AI Chain via the model input connection.
   - Connect the True branch of `Has Main Image?` to `Write Motion Prompt`.

6. **Initiate Sisif Video Generation:**
   - Add an **HTTP Request** node named `Generate Video from Image`.
   - Method: `POST`, URL: `https://sisif.ai/api/videos/generate/`.
   - Authentication: Generic Credential Type -> HTTP Bearer Auth (Sisif API).
   - Body parameters: `prompt` (`={{ $json.text.trim() }}`) and `image` (`={{ $('Extract Product Data').item.json.image_url }}`).
   - Connect `Write Motion Prompt` to `Generate Video from Image`.

7. **Build the Polling Loop:**
   - Add a **Wait** node named `Wait 30s` (duration: 30 seconds). Connect `Generate Video from Image` here.
   - Add an **HTTP Request** node named `Check Video Status` (GET, URL: `https://sisif.ai/api/videos/{{ $json.id }}/status/`, Bearer Auth). Connect `Wait 30s` here.
   - Add an **If** node named `Video Ready?` checking if `status === READY`. Connect `Check Video Status` here.
   - Add an **If** node named `Render Failed?` (Regex check on status for `^(ERROR|REFUNDED)$`). Connect the False branch of `Video Ready?` here.
   - Add an **If** node named `Max Attempts Reached?` checking if `{{ $runIndex + 1 }} >= 40`. Connect the False branch of `Render Failed?` here.
   - Connect the True branch of `Max Attempts Reached?` to a **Set** node named `Mark Timed Out` (sets `status` to `TIMEOUT`), and connect it to a Slack notification node.
   - Connect the False branch of `Max Attempts Reached?` back to `Wait 30s` to continue polling.

8. **Handle Video Download and Measurement:**
   - Add an **HTTP Request** node named `Download Video` (Response Format: File, URL: `={{ $json.video_url }}`). Connect the True branch of `Video Ready?` here.
   - Add a **Code** node named `Measure Video File` to calculate buffer length and generate a sanitized filename (`<title>-sisif.mp4`). Connect `Download Video` here.

9. **Create Shopify Staged Upload Target:**
   - Add an **HTTP Request** node named `Create Staged Upload` (POST to Shopify GraphQL endpoint, Shopify Access Token auth) using the `stagedUploadsCreate` GraphQL mutation. Connect `Measure Video File` here.
   - Add an **HTTP Request** node named `Download Video for Upload` (File download). Connect `Create Staged Upload` here.
   - Add a **Code** node named `Prepare Staged Upload` to parse returned targets and parameters. Connect `Download Video for Upload` here.
   - Add an **If** node named `Staged Target Ready?` (`upload_ready === true`). Connect `Prepare Staged Upload` here.

10. **Upload to Staged Target & Attach to Product:**
    - Add an **HTTP Request** node named `Upload to Staged Target` (POST, multipart form-data, mapping 12 dynamic parameter slots and the binary file `file`). Connect the True branch of `Staged Target Ready?` here.
    - Add an **If** node named `Upload Succeeded?` checking for HTTP 200–299 status and no errors. Connect `Upload to Staged Target` here.
    - Add an **HTTP Request** node named `Attach Video to Product` (POST to Shopify GraphQL endpoint using `productUpdate` mutation). Connect the True branch of `Upload Succeeded?` here.
    - Add an **If** node named `Attached to Shopify?` verifying zero GraphQL user errors. Connect `Attach Video to Product` here.
    - Connect the True branch to a **Slack** node (`Notify Slack: Attached`).

11. **Configure Google Drive Fallback:**
    - Add an **HTTP Request** node named `Download Video for Drive` (File download from Sisif URL). Connect this node from the failure branches of `Staged Target Ready?`, `Upload Succeeded?`, and `Attached to Shopify?`.
    - Add a **Google Drive** node named `Upload to Google Drive` (OAuth2 authentication, uploads file into `drive_folder_id`). Connect `Download Video for Drive` here.
    - Add a **Slack** node named `Notify Slack: Saved to Drive` to report fallback storage. Connect `Upload to Google Drive` here.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Create a Sisif API key | [Sisif API Keys](https://sisif.ai/users/api-keys/) |
| Shopify Custom App Setup | Requires `read_products`, `write_products`, and `write_files` scopes |
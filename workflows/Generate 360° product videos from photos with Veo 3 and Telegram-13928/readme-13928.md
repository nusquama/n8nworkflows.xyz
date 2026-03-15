Generate 360° product videos from photos with Veo 3 and Telegram

https://n8nworkflows.xyz/workflows/generate-360--product-videos-from-photos-with-veo-3-and-telegram-13928


# Generate 360° product videos from photos with Veo 3 and Telegram

## 1. Workflow Overview

This workflow turns a product photo sent to a Telegram bot into a generated 360° product video using Google Vertex AI Veo 3, then sends the resulting video back to the same Telegram user.

Its main use case is product visualization: a user submits a single image, and the workflow produces a cinematic orbit-style showcase video. The process includes input validation, Google Cloud authentication via a service account stored in Google Sheets, image preparation, Veo 3 submission, polling for completion, and Telegram delivery.

### 1.1 Input Reception and Validation
The workflow starts from a Telegram bot message. It verifies that the incoming message contains a photo and that the image meets a minimum size requirement.

### 1.2 Google Cloud Authentication
The workflow reads Google service account fields from Google Sheets, locally signs a JWT in a Code node, and exchanges that JWT for a Google OAuth access token.

### 1.3 Image Download and Conversion
Once authenticated, the workflow downloads the Telegram-hosted image file and converts it into Base64 so it can be embedded in the Vertex AI request payload.

### 1.4 Veo 3 Job Submission
The workflow builds a prompt and request payload for Veo 3, submits a long-running generation job to Vertex AI, and captures the returned operation name.

### 1.5 Polling and Timeout Handling
The workflow waits 2 minutes between checks and repeatedly polls Vertex AI until the operation is done or a timeout threshold is reached.

### 1.6 Video Delivery
When the video bytes are available, the workflow converts them into a binary file and sends the video back to the Telegram user.

---

## 2. Block-by-Block Analysis

## 2.1 Input Reception and Validation

**Overview:**  
This block listens for Telegram messages and verifies that the user sent a usable photo. It extracts core metadata such as chat ID, message ID, caption, and file ID for downstream processing.

**Nodes Involved:**  
- Telegram Trigger  
- 2. Validate Input

### Node: Telegram Trigger
- **Type and technical role:** `n8n-nodes-base.telegramTrigger` — entry point that receives Telegram `message` updates.
- **Configuration choices:**  
  - Configured to listen only to `message` updates.
  - Uses Telegram API credentials tied to the bot.
- **Key expressions or variables used:** None in node parameters.
- **Input and output connections:**  
  - No input; this is a workflow entry point.
  - Outputs to `2. Validate Input`.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases or potential failure types:**  
  - Telegram webhook misconfiguration
  - Invalid Telegram credentials
  - Bot not enabled or blocked by user
  - Updates other than standard messages are ignored
- **Sub-workflow reference:** None.

### Node: 2. Validate Input
- **Type and technical role:** `n8n-nodes-base.code` — custom JavaScript validation and normalization.
- **Configuration choices:**  
  - Reads `message` from the Telegram Trigger payload.
  - Fails if no message exists.
  - Fails if no `photo` array exists.
  - Uses the largest Telegram photo variant from `message.photo[message.photo.length - 1]`.
  - Rejects images where the smaller dimension is below 480 pixels.
  - Extracts:
    - `chatId`
    - `messageId`
    - `caption`
    - `file_id`
    - `error`
    - `errorMessage` if validation fails
- **Key expressions or variables used:**  
  - `message.photo[message.photo.length - 1]`
  - `Math.min(photo.width, photo.height) < 480`
- **Input and output connections:**  
  - Input from `Telegram Trigger`
  - Output to `1. Get Service Account Details`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**  
  - Text-only messages
  - Files sent as Telegram documents rather than photos
  - Missing `chat` or `message_id`
  - Very small images
- **Sub-workflow reference:** None.

**Important implementation note:**  
Although the sticky note says invalid input is rejected with a clear error, the current connection design does not branch on `error` immediately after validation. Instead, the workflow continues into the auth block. As built, validation errors may propagate until later and may not stop early in a clean way.

---

## 2.2 Google Cloud Authentication

**Overview:**  
This block retrieves service account fields from Google Sheets, signs a JWT using the private key, exchanges it for an OAuth token, and checks whether token retrieval succeeded.

**Nodes Involved:**  
- 1. Get Service Account Details  
- 2. Build JWT from Sheet  
- 3. Get Access Token  
- Check Auth Token Valid  
- Send Validation Error  
- Send Processing Message

### Node: 1. Get Service Account Details
- **Type and technical role:** `n8n-nodes-base.googleSheets` — fetches service account credential fields from a Google Sheet.
- **Configuration choices:**  
  - Reads from a spreadsheet placeholder identified as `YOUR_GOOGLE_SHEET_ID_HERE`.
  - Sheet selected as `gid=0`.
  - Uses Google Sheets OAuth2 credentials.
- **Key expressions or variables used:** None visible in parameters.
- **Input and output connections:**  
  - Input from `2. Validate Input`
  - Output to `2. Build JWT from Sheet`
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases or potential failure types:**  
  - Spreadsheet ID not replaced
  - OAuth permission issues
  - Missing expected columns: `client_email`, `private_key`, `project_id`, `scope`
  - Empty sheet or multiple unexpected rows
- **Sub-workflow reference:** None.

### Node: 2. Build JWT from Sheet
- **Type and technical role:** `n8n-nodes-base.code` — creates a signed JWT assertion from sheet-provided service account fields.
- **Configuration choices:**  
  - Imports Node.js `crypto`.
  - Uses:
    - `client_email`
    - `private_key`
    - `project_id`
    - `scope` with default `https://www.googleapis.com/auth/cloud-platform`
  - Replaces escaped newlines in the private key.
  - Builds JWT header and claims.
  - Signs with `RSA-SHA256`.
  - Returns `jwt`, `project_id`, `client_email`, and `error` state.
- **Key expressions or variables used:**  
  - `item.private_key.replace(/\\n/g, '\n')`
  - `now + 3600` for token lifetime
- **Input and output connections:**  
  - Input from `1. Get Service Account Details`
  - Output to `3. Get Access Token`
- **Version-specific requirements:** Type version `2`; requires Code node support for `require('crypto')`.
- **Edge cases or potential failure types:**  
  - Malformed PEM private key
  - Missing service account fields
  - Restricted Code-node environment
  - Signature generation errors
- **Sub-workflow reference:** None.

### Node: 3. Get Access Token
- **Type and technical role:** `n8n-nodes-base.httpRequest` — exchanges JWT for a Google OAuth2 access token.
- **Configuration choices:**  
  - `POST https://oauth2.googleapis.com/token`
  - Form URL encoded body:
    - `grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer`
    - `assertion={{ $json.jwt }}`
- **Key expressions or variables used:**  
  - `={{ $json.jwt }}`
- **Input and output connections:**  
  - Input from `2. Build JWT from Sheet`
  - Output to `Check Auth Token Valid`
- **Version-specific requirements:** Type version `4.2`.
- **Edge cases or potential failure types:**  
  - Invalid JWT
  - Expired or malformed claims
  - Incorrect OAuth scope
  - HTTP 400/401 responses
- **Sub-workflow reference:** None.

### Node: Check Auth Token Valid
- **Type and technical role:** `n8n-nodes-base.if` — checks whether `access_token` is present and non-empty.
- **Configuration choices:**  
  - Condition evaluates `={{ $json.access_token.isEmpty() }}` and checks that result is false.
  - True branch means token is considered valid.
  - False branch sends an error.
- **Key expressions or variables used:**  
  - `={{ $json.access_token.isEmpty() }}`
- **Input and output connections:**  
  - Input from `3. Get Access Token`
  - True output to `Send Processing Message`
  - False output to `Send Validation Error`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**  
  - If `access_token` is undefined, `.isEmpty()` may be brittle depending on runtime behavior
  - Upstream error payloads may not include `chatId` and `messageId`, making error messaging unreliable
- **Sub-workflow reference:** None.

### Node: Send Validation Error
- **Type and technical role:** `n8n-nodes-base.telegram` — sends an error message back to the user.
- **Configuration choices:**  
  - Text: `={{ $json.errorMessage }}`
  - Chat ID: `={{ $json.chatId }}`
  - Replies to original message via `messageId`
- **Key expressions or variables used:**  
  - `={{ $json.errorMessage }}`
  - `={{ $json.chatId }}`
  - `={{ $json.messageId }}`
- **Input and output connections:**  
  - Input from false branch of `Check Auth Token Valid`
  - No downstream nodes
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:**  
  - Missing `chatId` or `messageId`
  - Telegram send failure
  - Empty `errorMessage`
- **Sub-workflow reference:** None.

### Node: Send Processing Message
- **Type and technical role:** `n8n-nodes-base.telegram` — acknowledges processing has started.
- **Configuration choices:**  
  - Sends a user-facing status message:
    - “Creating your 360° product video...”
    - Estimated 3–4 minutes
    - Indicates Google Veo 3 processing
  - Replies to original message
- **Key expressions or variables used:**  
  - `={{ $json.chatId }}`
  - `={{ $json.messageId }}`
- **Input and output connections:**  
  - Input from true branch of `Check Auth Token Valid`
  - Output to `Download Image`
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:**  
  - Missing `chatId` or `messageId`
  - Telegram send failure
- **Sub-workflow reference:** None.

**Important implementation note:**  
There is a data continuity issue in this block: `1. Get Service Account Details` likely outputs only sheet row data and may overwrite fields created by `2. Validate Input`, such as `chatId`, `messageId`, and `caption`, unless the node is configured to preserve or merge incoming fields. Several later nodes assume those values still exist. This should be verified in n8n runtime.

---

## 2.3 Image Download and Conversion

**Overview:**  
This block downloads the selected Telegram image variant and converts the binary file into Base64 with MIME metadata for Vertex AI. If conversion fails, the user is notified.

**Nodes Involved:**  
- Download Image  
- Convert Image to Base64  
- Conversion OK?  
- Send Conversion Error

### Node: Download Image
- **Type and technical role:** `n8n-nodes-base.telegram` — downloads a Telegram file.
- **Configuration choices:**  
  - Resource: `file`
  - `fileId` is set to `={{ $json.result.reply_to_message.photo[3].file_id }}`
- **Key expressions or variables used:**  
  - `={{ $json.result.reply_to_message.photo[3].file_id }}`
- **Input and output connections:**  
  - Input from `Send Processing Message`
  - Output to `Convert Image to Base64`
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:**  
  - This expression depends on the output structure of `Send Processing Message`
  - It assumes exactly at least 4 photo sizes exist
  - It ignores the previously extracted `file_id`
  - Telegram API response shape may not include `result.reply_to_message.photo`
- **Sub-workflow reference:** None.

**Important implementation note:**  
This is one of the riskiest nodes in the workflow. A safer expression would typically use the validated file ID from upstream data, such as `{{$json.file_id}}`, rather than `reply_to_message.photo[3].file_id`.

### Node: Convert Image to Base64
- **Type and technical role:** `n8n-nodes-base.code` — converts binary image data to Base64 and normalizes MIME handling.
- **Configuration choices:**  
  - Iterates over all input items.
  - Tries multiple binary data shapes:
    - nested `.data`
    - Buffer
    - string
    - fallback through binary keys
  - Produces:
    - `imageBase64`
    - `imageMimeType`
    - `imageSize`
    - `conversionStatus`
  - On failure produces:
    - `error`
    - `conversionStatus: failed`
    - `conversionError`
- **Key expressions or variables used:**  
  - `item.binary.data`
  - fallback inspection through `Object.keys(item.binary)`
- **Input and output connections:**  
  - Input from `Download Image`
  - Output to `Conversion OK?`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**  
  - Telegram file download produced no binary data
  - Unexpected binary property name
  - Empty image buffer
  - MIME metadata missing or inaccurate
- **Sub-workflow reference:** None.

### Node: Conversion OK?
- **Type and technical role:** `n8n-nodes-base.if` — branches on conversion success.
- **Configuration choices:**  
  - Compares `conversionStatus` to `success`
- **Key expressions or variables used:**  
  - `={{ $json.conversionStatus }}`
- **Input and output connections:**  
  - Input from `Convert Image to Base64`
  - True output to `5. Prepare Veo Request`
  - False output to `Send Conversion Error`
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**  
  - Missing `conversionStatus`
- **Sub-workflow reference:** None.

### Node: Send Conversion Error
- **Type and technical role:** `n8n-nodes-base.telegram` — informs the user that image processing failed.
- **Configuration choices:**  
  - Sends detailed error with `conversionError`
  - Replies to original message
- **Key expressions or variables used:**  
  - `=❌ Failed to process your image: {{ $json.conversionError }}`
  - `={{ $json.chatId }}`
  - `={{ $json.messageId }}`
- **Input and output connections:**  
  - Input from false branch of `Conversion OK?`
  - No downstream nodes
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:**  
  - Missing `chatId`, `messageId`, or `conversionError`
- **Sub-workflow reference:** None.

---

## 2.4 Veo 3 Job Submission

**Overview:**  
This block assembles the generation prompt and Veo request body, submits the request to Vertex AI, and extracts the operation name used for asynchronous polling.

**Nodes Involved:**  
- 5. Prepare Veo Request  
- 6. Call Vertex AI Veo 3  
- Extract Operation Name

### Node: 5. Prepare Veo Request
- **Type and technical role:** `n8n-nodes-base.code` — builds the model prompt and API payload.
- **Configuration choices:**  
  - Base prompt requests:
    - 360-degree product showcase
    - smooth orbit camera
    - clean white background
    - centered product
    - cinematic movement
  - If Telegram caption exists, appends it as product context.
  - Payload includes:
    - `aspectRatio: 16:9`
    - `sampleCount: 1`
    - `durationSeconds: 8`
    - `personGeneration: allow_all`
    - `addWatermark: true`
    - `includeRaiReason: true`
    - `generateAudio: true`
  - Returns:
    - `api_payload` as a JSON string
    - `chatId`
    - `messageId`
    - `caption`
- **Key expressions or variables used:**  
  - `item.caption || ''`
  - `JSON.stringify(apiPayload)`
- **Input and output connections:**  
  - Input from `Conversion OK?` true branch
  - Output to `6. Call Vertex AI Veo 3`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**  
  - Missing `imageBase64`
  - Very large payload size
  - Unsupported MIME type
  - Caption containing prompt-injection-like content
- **Sub-workflow reference:** None.

### Node: 6. Call Vertex AI Veo 3
- **Type and technical role:** `n8n-nodes-base.httpRequest` — submits a long-running Veo generation request.
- **Configuration choices:**  
  - POST to:
    `https://us-central1-aiplatform.googleapis.com/v1/projects/{{project_id}}/locations/us-central1/publishers/google/models/veo-3.0-generate-preview:predictLongRunning`
  - Sends JSON body from `api_payload`
  - Headers:
    - `Authorization: Bearer {{ access_token }}`
    - `Content-Type: application/json`
- **Key expressions or variables used:**  
  - `{{ $('3. Get Access Token').item.json.project_id }}`
  - `{{ $('3. Get Access Token').item.json.access_token }}`
  - `={{ $json.api_payload }}`
- **Input and output connections:**  
  - Input from `5. Prepare Veo Request`
  - Output to `Extract Operation Name`
- **Version-specific requirements:** Type version `4.2`.
- **Edge cases or potential failure types:**  
  - Veo preview access not enabled
  - Invalid project or region
  - Invalid access token
  - Request schema mismatch
  - Quota, safety, or model availability errors
- **Sub-workflow reference:** None.

### Node: Extract Operation Name
- **Type and technical role:** `n8n-nodes-base.code` — extracts the long-running operation name from Vertex AI response.
- **Configuration choices:**  
  - If `response.name` exists, returns:
    - `operation_name`
    - `status: polling`
    - `poll_count: 0`
    - `chatId`
    - `messageId`
    - `caption`
  - Else returns failure information.
- **Key expressions or variables used:**  
  - `response.name`
- **Input and output connections:**  
  - Input from `6. Call Vertex AI Veo 3`
  - Output to `Wait 2 Minutes`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**  
  - Vertex AI returns an error object instead of `name`
  - Upstream HTTP node may not preserve request context fields
- **Sub-workflow reference:** None.

**Important implementation note:**  
`Extract Operation Name` expects `chatId`, `messageId`, and `caption` to exist on the HTTP response item. Depending on the HTTP Request node behavior, these values may not be preserved unless explicitly included or merged.

---

## 2.5 Polling and Timeout Handling

**Overview:**  
This block waits two minutes between status checks, polls the long-running Vertex AI operation, and either proceeds to delivery when the video is ready or loops again until a timeout occurs.

**Nodes Involved:**  
- Wait 2 Minutes  
- Poll Video Status  
- Is Video Ready?  
- Continue Polling  
- Check Timeout  
- Send Timeout Error

### Node: Wait 2 Minutes
- **Type and technical role:** `n8n-nodes-base.wait` — delay between polling attempts.
- **Configuration choices:**  
  - Wait duration: 2 minutes
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input from `Extract Operation Name`
  - Also loops back from `Check Timeout` false branch
  - Output to `Poll Video Status`
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:**  
  - Wait execution persistence depends on n8n setup
  - If wait/resume webhooks are not supported in the hosting setup, resumption can fail
- **Sub-workflow reference:** None.

### Node: Poll Video Status
- **Type and technical role:** `n8n-nodes-base.httpRequest` — checks operation status with Vertex AI.
- **Configuration choices:**  
  - POST to:
    `https://us-central1-aiplatform.googleapis.com/v1/projects/{{project_id}}/locations/us-central1/publishers/google/models/veo-3.0-generate-preview:fetchPredictOperation`
  - JSON body:
    - `operationName: {{ $json.operation_name }}`
- **Key expressions or variables used:**  
  - `{{ $('3. Get Access Token').item.json.project_id }}`
  - `{{ $json.operation_name }}`
- **Input and output connections:**  
  - Input from `Wait 2 Minutes`
  - Output to `Is Video Ready?`
- **Version-specific requirements:** Type version `4.2`.
- **Edge cases or potential failure types:**  
  - Missing Authorization header: this node currently does not explicitly set one
  - Expired token if the workflow runs too long
  - Invalid or missing `operation_name`
- **Sub-workflow reference:** None.

**Important implementation note:**  
Unlike the submission node, this polling node does not define an Authorization header. If the Vertex AI polling endpoint requires OAuth, this node will fail unless credentials are injected elsewhere or the API accepts unauthenticated polling, which is unlikely.

### Node: Is Video Ready?
- **Type and technical role:** `n8n-nodes-base.if` — checks whether the operation is complete.
- **Configuration choices:**  
  - Condition: `done === true`
- **Key expressions or variables used:**  
  - `={{ $json.done }}`
- **Input and output connections:**  
  - Input from `Poll Video Status`
  - True output to `Convert Video to File`
  - False output to `Continue Polling`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**  
  - Missing `done` property
  - Response schema changes from Vertex AI
- **Sub-workflow reference:** None.

### Node: Continue Polling
- **Type and technical role:** `n8n-nodes-base.code` — increments poll count and decides whether timeout is reached.
- **Configuration choices:**  
  - Increments `poll_count`
  - If `pollCount > 40`, returns timeout state
  - Otherwise returns:
    - `operation_name`
    - `poll_count`
    - `status: polling`
    - `chatId`
    - `messageId`
    - `caption`
- **Key expressions or variables used:**  
  - `(item.poll_count || 0) + 1`
  - `item.name || item.operation_name`
- **Input and output connections:**  
  - Input from false branch of `Is Video Ready?`
  - Output to `Check Timeout`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**  
  - Poll count may be lost if HTTP responses do not preserve original JSON fields
  - `operation_name` may be missing if relying on `item.name`
- **Sub-workflow reference:** None.

### Node: Check Timeout
- **Type and technical role:** `n8n-nodes-base.if` — checks whether the loop should stop with timeout.
- **Configuration choices:**  
  - Condition: `status == "timeout"`
- **Key expressions or variables used:**  
  - `={{ $json.status }}`
- **Input and output connections:**  
  - Input from `Continue Polling`
  - True output to `Send Timeout Error`
  - False output to `Wait 2 Minutes`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**  
  - Missing `status`
- **Sub-workflow reference:** None.

### Node: Send Timeout Error
- **Type and technical role:** `n8n-nodes-base.telegram` — sends timeout notification to the user.
- **Configuration choices:**  
  - Message includes the number of attempts from `poll_count`
  - Replies to original message
- **Key expressions or variables used:**  
  - `{{ $json.poll_count }}`
  - `={{ $json.chatId }}`
  - `={{ $json.messageId }}`
- **Input and output connections:**  
  - Input from true branch of `Check Timeout`
  - No downstream nodes
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:**  
  - Missing `chatId`, `messageId`, or `poll_count`
- **Sub-workflow reference:** None.

---

## 2.6 Video Delivery

**Overview:**  
This block converts the Base64 video returned by Vertex AI into a binary file and sends it to the Telegram chat.

**Nodes Involved:**  
- Convert Video to File  
- Send Video to User

### Node: Convert Video to File
- **Type and technical role:** `n8n-nodes-base.convertToFile` — converts Base64 video bytes into binary output.
- **Configuration choices:**  
  - Operation: `toBinary`
  - Source property: `response.videos[0].bytesBase64Encoded`
- **Key expressions or variables used:**  
  - `response.videos[0].bytesBase64Encoded`
- **Input and output connections:**  
  - Input from true branch of `Is Video Ready?`
  - Output to `Send Video to User`
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:**  
  - Missing `response.videos[0]`
  - Unexpected Vertex AI output schema
  - Empty or invalid Base64 data
- **Sub-workflow reference:** None.

### Node: Send Video to User
- **Type and technical role:** `n8n-nodes-base.telegram` — sends the generated video as Telegram media.
- **Configuration choices:**  
  - Operation: `sendVideo`
  - Uses binary data
  - Chat ID is taken from the original trigger:
    `={{ $('Telegram Trigger').item.json.message.chat.id }}`
- **Key expressions or variables used:**  
  - `={{ $('Telegram Trigger').item.json.message.chat.id }}`
- **Input and output connections:**  
  - Input from `Convert Video to File`
  - No downstream nodes
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:**  
  - Cross-node item reference may fail if execution context changes after Wait node resume
  - Large file may exceed Telegram limits
  - Missing binary property name if Convert to File uses a different default output key
- **Sub-workflow reference:** None.

**Important implementation note:**  
This node references the original `Telegram Trigger` item directly instead of using `chatId` carried through the workflow. After a Wait node resume, direct item references can be fragile. Using `{{$json.chatId}}` would usually be more reliable.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Telegram Trigger | Telegram Trigger | Receives Telegram message updates from the bot |  | 2. Validate Input | ## Receive & Validate<br>Listens for Telegram messages and checks that a photo was sent and is at least 480px. Rejects documents or text-only messages and replies with a clear error. |
| 2. Validate Input | Code | Validates message content and extracts photo/chat metadata | Telegram Trigger | 1. Get Service Account Details | ## Receive & Validate<br>Listens for Telegram messages and checks that a photo was sent and is at least 480px. Rejects documents or text-only messages and replies with a clear error. |
| 1. Get Service Account Details | Google Sheets | Reads service account fields from a Google Sheet | 2. Validate Input | 2. Build JWT from Sheet | ## Google Cloud Auth<br>Reads Service Account credentials from Google Sheets, signs a JWT locally, and exchanges it for a short-lived OAuth access token. Runs fresh on every request. |
| 2. Build JWT from Sheet | Code | Builds and signs a Google service account JWT locally | 1. Get Service Account Details | 3. Get Access Token | ## Google Cloud Auth<br>Reads Service Account credentials from Google Sheets, signs a JWT locally, and exchanges it for a short-lived OAuth access token. Runs fresh on every request. |
| 3. Get Access Token | HTTP Request | Exchanges JWT assertion for Google OAuth access token | 2. Build JWT from Sheet | Check Auth Token Valid | ## Google Cloud Auth<br>Reads Service Account credentials from Google Sheets, signs a JWT locally, and exchanges it for a short-lived OAuth access token. Runs fresh on every request. |
| Check Auth Token Valid | If | Branches based on whether OAuth token retrieval succeeded | 3. Get Access Token | Send Processing Message, Send Validation Error | ## Google Cloud Auth<br>Reads Service Account credentials from Google Sheets, signs a JWT locally, and exchanges it for a short-lived OAuth access token. Runs fresh on every request. |
| Send Validation Error | Telegram | Sends an error message to the Telegram user | Check Auth Token Valid |  |  |
| Send Processing Message | Telegram | Notifies the user that video generation has started | Check Auth Token Valid | Download Image | ## Download & Convert<br>Downloads the highest-resolution version of the photo from Telegram and converts it to Base64. Any conversion failure is caught and reported back to the user. |
| Download Image | Telegram | Downloads the Telegram photo file | Send Processing Message | Convert Image to Base64 | ## Download & Convert<br>Downloads the highest-resolution version of the photo from Telegram and converts it to Base64. Any conversion failure is caught and reported back to the user. |
| Convert Image to Base64 | Code | Converts binary image content into Base64 and MIME metadata | Download Image | Conversion OK? | ## Download & Convert<br>Downloads the highest-resolution version of the photo from Telegram and converts it to Base64. Any conversion failure is caught and reported back to the user. |
| Conversion OK? | If | Branches on image conversion success or failure | Convert Image to Base64 | 5. Prepare Veo Request, Send Conversion Error | ## Download & Convert<br>Downloads the highest-resolution version of the photo from Telegram and converts it to Base64. Any conversion failure is caught and reported back to the user. |
| Send Conversion Error | Telegram | Sends image-processing failure message to the user | Conversion OK? |  | ## Download & Convert<br>Downloads the highest-resolution version of the photo from Telegram and converts it to Base64. Any conversion failure is caught and reported back to the user. |
| 5. Prepare Veo Request | Code | Builds prompt and JSON payload for Veo 3 | Conversion OK? | 6. Call Vertex AI Veo 3 | ## Submit to Veo 3<br>Builds the API payload with a cinematic 360° orbit prompt and submits a long-running job to Vertex AI Veo 3. Gets back an operation ID used for polling. |
| 6. Call Vertex AI Veo 3 | HTTP Request | Submits long-running Veo generation request to Vertex AI | 5. Prepare Veo Request | Extract Operation Name | ## Submit to Veo 3<br>Builds the API payload with a cinematic 360° orbit prompt and submits a long-running job to Vertex AI Veo 3. Gets back an operation ID used for polling. |
| Extract Operation Name | Code | Extracts polling operation name from Vertex AI response | 6. Call Vertex AI Veo 3 | Wait 2 Minutes | ## Poll for Result<br>Checks every 2 minutes whether the video is ready. Times out after 40 attempts (~10 min) and notifies the user with a friendly error if generation takes too long. |
| Wait 2 Minutes | Wait | Delays the next polling attempt by 2 minutes | Extract Operation Name, Check Timeout | Poll Video Status | ## Poll for Result<br>Checks every 2 minutes whether the video is ready. Times out after 40 attempts (~10 min) and notifies the user with a friendly error if generation takes too long. |
| Poll Video Status | HTTP Request | Polls Vertex AI for long-running operation status | Wait 2 Minutes | Is Video Ready? | ## Poll for Result<br>Checks every 2 minutes whether the video is ready. Times out after 40 attempts (~10 min) and notifies the user with a friendly error if generation takes too long. |
| Is Video Ready? | If | Checks whether the operation has completed | Poll Video Status | Convert Video to File, Continue Polling | ## Poll for Result<br>Checks every 2 minutes whether the video is ready. Times out after 40 attempts (~10 min) and notifies the user with a friendly error if generation takes too long. |
| Continue Polling | Code | Increments poll count and emits timeout or next polling state | Is Video Ready? | Check Timeout | ## Poll for Result<br>Checks every 2 minutes whether the video is ready. Times out after 40 attempts (~10 min) and notifies the user with a friendly error if generation takes too long. |
| Check Timeout | If | Branches between timeout handling and another polling cycle | Continue Polling | Send Timeout Error, Wait 2 Minutes | ## Poll for Result<br>Checks every 2 minutes whether the video is ready. Times out after 40 attempts (~10 min) and notifies the user with a friendly error if generation takes too long. |
| Send Timeout Error | Telegram | Informs the user that generation timed out | Check Timeout |  | ## Poll for Result<br>Checks every 2 minutes whether the video is ready. Times out after 40 attempts (~10 min) and notifies the user with a friendly error if generation takes too long. |
| Convert Video to File | Convert to File | Converts returned Base64 video into binary file data | Is Video Ready? | Send Video to User | ## Deliver Video<br>Converts the Base64 video bytes to a file and sends it directly to the user in Telegram. |
| Send Video to User | Telegram | Sends the generated video back to the Telegram user | Convert Video to File |  | ## Deliver Video<br>Converts the Base64 video bytes to a file and sends it directly to the user in Telegram. |
| 📋 Overview | Sticky Note | Workspace documentation and setup instructions |  |  | ## 🎬 360° Product Video Generator<br><br>Turn a single product photo into a cinematic 360° video using Google Veo 3 — delivered straight to Telegram.<br><br>## How it works<br>1. User sends a product photo to your Telegram bot<br>2. The workflow authenticates with Google Cloud using a Service Account stored in Google Sheets<br>3. The image is sent to Vertex AI (Veo 3) with a 360° orbit camera prompt<br>4. The workflow polls every 2 minutes until the video is ready (up to 10 min)<br>5. The finished video is sent back to the user in Telegram<br><br>## Setup steps<br>1. Create a Telegram bot via @BotFather and add the credentials in n8n<br>2. Enable the **Vertex AI API** in your Google Cloud project and request Veo 3 preview access<br>3. Create a Service Account with `roles/aiplatform.user` and download the JSON key<br>4. Paste the key fields into your Google Sheet — columns needed: `client_email`, `private_key`, `project_id`, `scope`<br>5. Update the **1. Get Service Account Details** node with your Sheet ID<br>6. Connect your Google and Telegram credentials, then activate the workflow |
| Group 1 | Sticky Note | Visual group note for input validation block |  |  | ## Receive & Validate<br>Listens for Telegram messages and checks that a photo was sent and is at least 480px. Rejects documents or text-only messages and replies with a clear error. |
| Group 2 | Sticky Note | Visual group note for auth block |  |  | ## Google Cloud Auth<br>Reads Service Account credentials from Google Sheets, signs a JWT locally, and exchanges it for a short-lived OAuth access token. Runs fresh on every request. |
| Group 3 | Sticky Note | Visual group note for image processing block |  |  | ## Download & Convert<br>Downloads the highest-resolution version of the photo from Telegram and converts it to Base64. Any conversion failure is caught and reported back to the user. |
| Group 4 | Sticky Note | Visual group note for Veo submission block |  |  | ## Submit to Veo 3<br>Builds the API payload with a cinematic 360° orbit prompt and submits a long-running job to Vertex AI Veo 3. Gets back an operation ID used for polling. |
| Group 5 | Sticky Note | Visual group note for polling block |  |  | ## Poll for Result<br>Checks every 2 minutes whether the video is ready. Times out after 40 attempts (~10 min) and notifies the user with a friendly error if generation takes too long. |
| Group 6 | Sticky Note | Visual group note for delivery block |  |  | ## Deliver Video<br>Converts the Base64 video bytes to a file and sends it directly to the user in Telegram. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it something like:  
   **Generate 360° product videos from photos with Veo 3 and Telegram**.

2. **Create a Telegram Trigger node**
   - Node type: `Telegram Trigger`
   - Configure Telegram credentials using a bot created via `@BotFather`
   - Set update type to `message`
   - This is the main entry point

3. **Add a Code node named `2. Validate Input`**
   - Connect it after `Telegram Trigger`
   - Use JavaScript to:
     - verify `message` exists
     - verify `message.photo` exists and is a non-empty array
     - select the largest photo variant
     - reject images smaller than 480 px on the shortest side
     - extract:
       - `chatId`
       - `messageId`
       - `caption`
       - `file_id`
     - add `error` and `errorMessage` fields when validation fails
   - Use the same validation logic as in the workflow JSON

4. **Add a Google Sheets node named `1. Get Service Account Details`**
   - Connect it after `2. Validate Input`
   - Configure Google Sheets OAuth2 credentials
   - Select the spreadsheet containing service account fields
   - Select the target sheet/tab
   - Ensure the sheet contains columns:
     - `client_email`
     - `private_key`
     - `project_id`
     - `scope`
   - Replace the placeholder sheet ID with your real Google Sheet ID

5. **Add a Code node named `2. Build JWT from Sheet`**
   - Connect it after `1. Get Service Account Details`
   - In JavaScript:
     - import `crypto`
     - read `client_email`, `private_key`, `project_id`, `scope`
     - replace escaped `\n` in the private key with actual newlines
     - build JWT header and claims
     - sign with `RSA-SHA256`
     - output:
       - `jwt`
       - `project_id`
       - `client_email`
       - `error`
       - `errorMessage` on failure

6. **Add an HTTP Request node named `3. Get Access Token`**
   - Connect it after `2. Build JWT from Sheet`
   - Method: `POST`
   - URL: `https://oauth2.googleapis.com/token`
   - Content type: `Form URL Encoded`
   - Add body parameters:
     - `grant_type` = `urn:ietf:params:oauth:grant-type:jwt-bearer`
     - `assertion` = `{{$json.jwt}}`

7. **Add an If node named `Check Auth Token Valid`**
   - Connect it after `3. Get Access Token`
   - Condition should verify that `access_token` is not empty
   - True branch = continue
   - False branch = send error

8. **Add a Telegram node named `Send Validation Error`**
   - Connect it to the false output of `Check Auth Token Valid`
   - Operation: send message
   - Chat ID: `{{$json.chatId}}`
   - Text: `{{$json.errorMessage}}`
   - Set reply-to message ID to `{{$json.messageId}}`

9. **Add a Telegram node named `Send Processing Message`**
   - Connect it to the true output of `Check Auth Token Valid`
   - Operation: send message
   - Chat ID: `{{$json.chatId}}`
   - Text should indicate generation has started, for example:
     - creating the 360° video
     - estimated wait time
     - processing with Veo 3
   - Set reply-to message ID to `{{$json.messageId}}`

10. **Add a Telegram node named `Download Image`**
    - Connect it after `Send Processing Message`
    - Resource: `file`
    - Provide a file ID expression
    - The current workflow uses:
      `{{$json.result.reply_to_message.photo[3].file_id}}`
    - For a more robust rebuild, prefer carrying forward and using:
      `{{$json.file_id}}`
    - Ensure binary download is enabled as needed by your n8n version

11. **Add a Code node named `Convert Image to Base64`**
    - Connect it after `Download Image`
    - Implement logic to:
      - detect binary payload shape
      - convert image bytes to Base64
      - capture MIME type
      - return `conversionStatus = success`
      - on error return `conversionStatus = failed` and `conversionError`

12. **Add an If node named `Conversion OK?`**
    - Connect it after `Convert Image to Base64`
    - Condition:
      - `conversionStatus` equals `success`
    - True branch = continue to Veo request
    - False branch = send conversion error

13. **Add a Telegram node named `Send Conversion Error`**
    - Connect it to the false output of `Conversion OK?`
    - Operation: send message
    - Chat ID: `{{$json.chatId}}`
    - Text should include `{{$json.conversionError}}`
    - Set reply-to message ID to `{{$json.messageId}}`

14. **Add a Code node named `5. Prepare Veo Request`**
    - Connect it to the true output of `Conversion OK?`
    - Build a prompt describing:
      - a professional 360-degree product showcase
      - smooth orbit motion
      - white background
      - centered product
      - cinematic movement
    - Append Telegram caption as additional context if present
    - Build payload:
      - `instances[0].prompt`
      - `instances[0].image.bytesBase64Encoded`
      - `instances[0].image.mimeType`
      - `parameters.aspectRatio = 16:9`
      - `parameters.sampleCount = 1`
      - `parameters.durationSeconds = 8`
      - `parameters.personGeneration = allow_all`
      - `parameters.addWatermark = true`
      - `parameters.includeRaiReason = true`
      - `parameters.generateAudio = true`
    - Return:
      - serialized JSON payload
      - `chatId`
      - `messageId`
      - `caption`

15. **Add an HTTP Request node named `6. Call Vertex AI Veo 3`**
    - Connect it after `5. Prepare Veo Request`
    - Method: `POST`
    - URL:
      `https://us-central1-aiplatform.googleapis.com/v1/projects/{{$('3. Get Access Token').item.json.project_id}}/locations/us-central1/publishers/google/models/veo-3.0-generate-preview:predictLongRunning`
    - Send JSON body using the prepared payload
    - Add headers:
      - `Authorization` = `Bearer {{$('3. Get Access Token').item.json.access_token}}`
      - `Content-Type` = `application/json`

16. **Add a Code node named `Extract Operation Name`**
    - Connect it after `6. Call Vertex AI Veo 3`
    - Read the API response
    - If `name` exists:
      - set `operation_name`
      - set `status = polling`
      - set `poll_count = 0`
      - carry `chatId`, `messageId`, `caption`
    - Otherwise mark the item as failed

17. **Add a Wait node named `Wait 2 Minutes`**
    - Connect it after `Extract Operation Name`
    - Wait time: `2 minutes`

18. **Add an HTTP Request node named `Poll Video Status`**
    - Connect it after `Wait 2 Minutes`
    - Method: `POST`
    - URL:
      `https://us-central1-aiplatform.googleapis.com/v1/projects/{{$('3. Get Access Token').item.json.project_id}}/locations/us-central1/publishers/google/models/veo-3.0-generate-preview:fetchPredictOperation`
    - JSON body:
      - `operationName` = `{{$json.operation_name}}`
    - Add the same Authorization header as the submit node if required:
      - `Authorization` = `Bearer {{$('3. Get Access Token').item.json.access_token}}`
      - `Content-Type` = `application/json`

19. **Add an If node named `Is Video Ready?`**
    - Connect it after `Poll Video Status`
    - Condition:
      - `done` equals `true`
    - True branch = convert and send video
    - False branch = continue polling

20. **Add a Code node named `Continue Polling`**
    - Connect it to the false output of `Is Video Ready?`
    - Increment `poll_count`
    - If count exceeds `40`, output:
      - `status = timeout`
      - `chatId`
      - `messageId`
    - Otherwise output:
      - `operation_name`
      - `poll_count`
      - `status = polling`
      - `chatId`
      - `messageId`
      - `caption`

21. **Add an If node named `Check Timeout`**
    - Connect it after `Continue Polling`
    - Condition:
      - `status` equals `timeout`
    - True branch = send timeout error
    - False branch = loop back to `Wait 2 Minutes`

22. **Connect the false output of `Check Timeout` back to `Wait 2 Minutes`**
    - This forms the polling loop.

23. **Add a Telegram node named `Send Timeout Error`**
    - Connect it to the true output of `Check Timeout`
    - Operation: send message
    - Chat ID: `{{$json.chatId}}`
    - Text should mention timeout and include `{{$json.poll_count}}`
    - Set reply-to message ID to `{{$json.messageId}}`

24. **Add a Convert to File node named `Convert Video to File`**
    - Connect it to the true output of `Is Video Ready?`
    - Operation: `toBinary`
    - Source property:
      `response.videos[0].bytesBase64Encoded`

25. **Add a Telegram node named `Send Video to User`**
    - Connect it after `Convert Video to File`
    - Operation: `sendVideo`
    - Enable binary data sending
    - Current workflow uses chat ID from:
      `{{$('Telegram Trigger').item.json.message.chat.id}}`
    - For a more robust rebuild, prefer using:
      `{{$json.chatId}}`
      if you preserve it through the polling chain

26. **Add workspace sticky notes if desired**
    - One overview note with setup requirements
    - Group notes for:
      - Receive & Validate
      - Google Cloud Auth
      - Download & Convert
      - Submit to Veo 3
      - Poll for Result
      - Deliver Video

27. **Configure required credentials**
    - **Telegram API credentials**
      - Connect the same bot credentials to:
        - Telegram Trigger
        - Send Validation Error
        - Send Processing Message
        - Download Image
        - Send Conversion Error
        - Send Timeout Error
        - Send Video to User
    - **Google Sheets OAuth2 credentials**
      - Connect them to `1. Get Service Account Details`

28. **Prepare Google Cloud outside n8n**
    - Enable Vertex AI API
    - Request access to Veo 3 preview if required
    - Create a Service Account with role:
      - `roles/aiplatform.user`
    - Download the JSON key
    - Put these values into Google Sheets:
      - `client_email`
      - `private_key`
      - `project_id`
      - `scope`

29. **Test with a Telegram photo**
    - Send a photo, not a document
    - Prefer image sizes above 1024×1024 for better results
    - Verify:
      - processing message is sent
      - image downloads correctly
      - Vertex AI accepts the prompt
      - polling works
      - final video is sent

30. **Recommended hardening before production**
    - Branch invalid input immediately after `2. Validate Input`
    - Preserve `chatId`, `messageId`, `caption`, and `file_id` across Google Sheets and HTTP nodes
    - Use `file_id` directly in `Download Image`
    - Add Authorization headers to `Poll Video Status`
    - Add explicit error handling for Vertex AI request failures
    - Use `{{$json.chatId}}` for final delivery rather than referencing the trigger node

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Turn a single product photo into a cinematic 360° video using Google Veo 3 — delivered straight to Telegram. | Workflow overview |
| Create a Telegram bot via @BotFather and add the credentials in n8n. | Telegram setup |
| Enable the Vertex AI API in your Google Cloud project and request Veo 3 preview access. | Google Cloud setup |
| Create a Service Account with `roles/aiplatform.user` and download the JSON key. | IAM setup |
| Paste the key fields into your Google Sheet — columns needed: `client_email`, `private_key`, `project_id`, `scope`. | Credential storage design |
| Update the `1. Get Service Account Details` node with your Sheet ID. | Google Sheets node setup |
| Connect your Google and Telegram credentials, then activate the workflow. | Deployment step |
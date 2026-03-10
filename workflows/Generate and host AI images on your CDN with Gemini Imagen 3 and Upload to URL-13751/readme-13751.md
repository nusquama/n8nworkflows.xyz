Generate and host AI images on your CDN with Gemini Imagen 3 and Upload to URL

https://n8nworkflows.xyz/workflows/generate-and-host-ai-images-on-your-cdn-with-gemini-imagen-3-and-upload-to-url-13751


# Generate and host AI images on your CDN with Gemini Imagen 3 and Upload to URL

### 1. Workflow Overview

This workflow automates the generation of high-quality AI images using **Google Gemini Imagen 3** and immediately hosts the resulting files on a Content Delivery Network (CDN). It is designed to handle two distinct business use cases through a single entry point: translating marketing assets (Localization) and creating photorealistic product mockups.

The workflow follows a structured path:
1.  **Entry & Routing**: Receives a JSON payload via Webhook and directs it based on the requested job type.
2.  **Prompt Engineering**: Uses custom JavaScript logic to craft precise, high-fidelity prompts for the Imagen 3 model.
3.  **AI Generation**: Interfaces with Google’s Generative Language API.
4.  **Binary Processing**: Converts the AI-generated Base64 data into standard n8n binary objects.
5.  **CDN Integration**: Uploads the image to a pre-defined CDN URL using the `Upload to URL` node.
6.  **Response Delivery**: Constructs and returns a structured JSON response containing the public URL of the hosted image.

---

### 2. Block-by-Block Analysis

#### 2.1 Entry & Routing
**Overview:** This block acts as the gateway for the workflow, receiving external requests and determining which processing pipeline to trigger.

*   **Nodes Involved:**
    *   `Webhook – Receive Image Job`
    *   `Route by Job Type` (Switch)
    *   `Respond – Error`
*   **Node Details:**
    *   **Webhook – Receive Image Job:** A Webhook node configured for `POST` requests at the `/gemini-image-job` path. It expects a JSON body containing `jobType` and specific parameters like `targetLanguage` or `productType`.
    *   **Route by Job Type:** A Switch node that evaluates `$json.body.jobType`.
        *   **Localize Path:** Triggered if the value is "localize".
        *   **Mockup Path:** Triggered if the value is "mockup".
        *   **Fallback:** Redirects to an error response node if no match is found.
    *   **Respond – Error:** A Webhook Response node that returns the input JSON (usually containing error details) if an invalid job type is provided.

#### 2.2 Localised Marketing Campaign Pipeline
**Overview:** This block generates a marketing image where the text content is translated while maintaining brand consistency.

*   **Nodes Involved:**
    *   `Build Localize Prompt`
    *   `Gemini Imagen 3 – Localize`
    *   `Decode Localize Image`
    *   `Upload to URL`
    *   `Build Localize Response`
    *   `Respond to Webhook – Localize`
*   **Node Details:**
    *   **Build Localize Prompt:** A Code node that takes `targetLanguage`, `brandName`, and `campaignText` to generate a prompt focused on layout preservation and typography. It also generates a unique `jobId`.
    *   **Gemini Imagen 3 – Localize:** An HTTP Request node.
        *   **Endpoint:** `https://generativelanguage.googleapis.com/v1beta/models/imagen-3.0-generate-001:predict`
        *   **Auth:** Uses Header Auth (`x-goog-api-key`).
    *   **Decode Localize Image:** A Code node that takes the `bytesBase64Encoded` string from Gemini and converts it into a Buffer. It outputs an n8n binary object with the correct MIME type (`image/png`).
    *   **Upload to URL:** A specialized node that performs a `PUT` request to a presigned CDN URL to host the binary file.
    *   **Build Localize Response:** A Code node that formats the final metadata, including the hardcoded `YOUR_CDN_DOMAIN`.

#### 2.3 High-Fidelity Product Mockup Pipeline
**Overview:** This block generates photorealistic product images based on descriptions and color schemes.

*   **Nodes Involved:**
    *   `Build Mockup Prompt`
    *   `Gemini Imagen 3 – Mockup`
    *   `Decode Mockup Image`
    *   `Upload to URL1`
    *   `Build Mockup Response`
    *   `Respond to Webhook – Mockup`
*   **Node Details:**
    *   **Build Mockup Prompt:** A Code node creating a prompt for a `productType` with a specific `logoDescription` and `colorScheme`. It emphasizes studio lighting and texture.
    *   **Gemini Imagen 3 – Mockup:** Mirror of the Localize HTTP node, calling the same Google endpoint but with the mockup-specific prompt.
    *   **Decode Mockup Image:** Converts the Base64 output into a binary file named `mockup_{product}_{jobId}.png`.
    *   **Upload to URL1:** Handles the binary upload to the CDN presigned URL for the mockup branch.
    *   **Build Mockup Response:** Returns a success JSON including the dynamic `publicUrl`.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Webhook – Receive Image Job | Webhook | Entry Point | (None) | Route by Job Type | Webhook – Receive Image Job accepts a POST at /gemini-image-job |
| Route by Job Type | Switch | Logic Router | Webhook | Build Localize Prompt, Build Mockup Prompt, Respond - Error | Routes: Output 0 localize, Output 1 mockup |
| Respond – Error | Webhook Response | Error Handler | Route by Job Type | (None) | Fallback → Respond – Error node returns a 400 JSON error |
| Build Localize Prompt | Code | Prompt Engineering | Route by Job Type | Gemini Imagen 3 – Localize | Injects brandName and campaignText from the payload. |
| Gemini Imagen 3 – Localize | HTTP Request | AI Generation | Build Localize Prompt | Decode Localize Image | Returns predictions[0].bytesBase64Encoded. |
| Decode Localize Image | Code | Data Conversion | Gemini Imagen 3 – Localize | Upload to URL | Decodes base64 → n8n binary (mimeType image/png). |
| Upload to URL | Upload to URL | CDN Hosting | Decode Localize Image | Build Localize Response | PUTs the binary to your CDN presigned URL. |
| Build Localize Response | Code | Response Prep | Upload to URL | Respond to Webhook – Localize | Returns { success, jobId, language, publicUrl... }. |
| Respond to Webhook – Localize | Webhook Response | Delivery | Build Localize Response | (None) | Sends the JSON back to the caller. |
| Build Mockup Prompt | Code | Prompt Engineering | Route by Job Type | Gemini Imagen 3 – Mockup | Photorealistic productType with brand logo/pattern. |
| Gemini Imagen 3 – Mockup | HTTP Request | AI Generation | Build Mockup Prompt | Decode Mockup Image | Returns the photorealistic product image as base64. |
| Decode Mockup Image | Code | Data Conversion | Gemini Imagen 3 – Mockup | Upload to URL1 | Decodes base64 → binary. |
| Upload to URL1 | Upload to URL | CDN Hosting | Decode Mockup Image | Build Mockup Response | PUTs the binary to your CDN presigned URL. |
| Build Mockup Response | Code | Response Prep | Upload to URL1 | Respond to Webhook – Mockup | Returns { success, jobId, productType, publicUrl... }. |
| Respond to Webhook – Mockup | Webhook Response | Delivery | Build Mockup Response | (None) | Sends the JSON back to the caller. |

---

### 4. Reproducing the Workflow from Scratch

1.  **Trigger Setup**: Create a **Webhook** node. Set method to `POST` and path to `gemini-image-job`. Set HTTP Response Mode to "When Last Node Finishes".
2.  **Logic Routing**: Add a **Switch** node. Use the expression `{{ $json.body.jobType.toLowerCase() }}`. Create two routing rules for "localize" and "mockup".
3.  **Prompt Construction**: 
    *   Create a **Code** node for the "Localize" branch. Use JavaScript to extract `targetLanguage`, `brandName`, and `campaignText` from `$json.body`. Generate a `jobId` using `Date.now()`.
    *   Create a **Code** node for the "Mockup" branch. Extract `productType`, `colorScheme`, and `logoDescription`.
4.  **AI Integration**: 
    *   Add an **HTTP Request** node. Method: `POST`. URL: `https://generativelanguage.googleapis.com/v1beta/models/imagen-3.0-generate-001:predict`.
    *   **Authentication**: Create "Header Auth" with `x-goog-api-key` as the name and your Gemini API Key as the value.
    *   Repeat for the second branch.
5.  **Binary Processing**:
    *   Add a **Code** node to both branches. Use `Buffer.from(predictions[0].bytesBase64Encoded, 'base64')` to create a buffer. 
    *   Return the data in the `binary` property of the n8n item with `mimeType: 'image/png'`.
6.  **File Hosting**: 
    *   Add the **Upload to URL** node. Configure your presigned PUT URL and credentials. 
    *   Note: Ensure the destination CDN supports `PUT` requests for binary data.
7.  **Final Output**:
    *   Add a **Code** node to construct the final JSON object. Ensure you replace `YOUR_CDN_DOMAIN` with your actual domain (e.g., `https://cdn.mycompany.com/`).
    *   Add a **Webhook Response** node at the end of each branch to return the final JSON.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Obtain Gemini API Keys | [Google AI Studio](https://aistudio.google.com) |
| Required API Endpoint | `POST https://generativelanguage.googleapis.com/v1beta/models/imagen-3.0-generate-001:predict` |
| Manual Action Required | Replace `YOUR_CDN_DOMAIN` in "Build Response" nodes. |
| Manual Action Required | Configure `YOUR_CDN_PRESIGNED_PUT_URL` in Upload to URL nodes. |
| Auth Configuration | Use `Google AI Header Auth` - Header name: `x-goog-api-key`. |
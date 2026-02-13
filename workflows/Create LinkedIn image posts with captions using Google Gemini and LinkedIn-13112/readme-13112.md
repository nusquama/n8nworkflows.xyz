Create LinkedIn image posts with captions using Google Gemini and LinkedIn

https://n8nworkflows.xyz/workflows/create-linkedin-image-posts-with-captions-using-google-gemini-and-linkedin-13112


# Create LinkedIn image posts with captions using Google Gemini and LinkedIn

Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow generates **a LinkedIn-ready image + caption** from a user-submitted quote (via a website form) using **Google Gemini**, returns the result to the website (for preview/edit), then accepts a second webhook call to **publish the final post to LinkedIn**.

**Target use cases:**
- Marketing/branding teams generating quote-style LinkedIn creatives
- Solo creators producing consistent visual posts quickly
- Internal comms creating professional micro-content from short prompts

### 1.1 Input Reception (Website → n8n)
- Receives a quote from a web form via a webhook endpoint.

### 1.2 AI Generation (Gemini Image + Gemini Caption)
- Branch A: Generates a stylized LinkedIn image (Gemini image model), then converts it to base64 for easy transport to a website.
- Branch B: Generates a LinkedIn caption using a Gemini chat model (via LangChain Agent).

### 1.3 Packaging & Response (n8n → Website)
- Merges image + caption into one JSON payload and returns it to the website.

### 1.4 Publishing (Website → n8n → LinkedIn)
- Receives a second webhook request containing the chosen final content and posts it to LinkedIn.
- Returns a success JSON response with CORS headers.

---

## 2. Block-by-Block Analysis

### Block 1 — Website Form Intake
**Overview:** Receives the user’s quote from an external website and fans out to two parallel AI-generation branches (image + caption).  
**Nodes involved:** `Webhook - Receive Form`

#### Node: Webhook - Receive Form
- **Type / Role:** `Webhook` — HTTP entry point for quote submissions.
- **Configuration choices:**
  - **Path:** `Linkedin-update`
  - **Method:** `POST`
  - **Response mode:** `Last Node` (the workflow returns whatever the last node outputs in that execution path)
  - **Raw body enabled:** `options.rawBody = true` (keeps the unparsed raw request body available; useful for signature verification or debugging).
- **Key expressions/variables:**
  - Downstream nodes reference `{{$json.body.quote}}` → implies the incoming request is expected to parse into `body.quote`.
- **Connections:**
  - Output splits to:
    - `Generate post image`
    - `Generate post caption`
- **Edge cases / failures:**
  - If the incoming payload doesn’t include `body.quote`, both AI prompts will degrade or error (empty prompt).
  - Because `rawBody` is enabled, ensure your webhook content-type is correct (typically `application/json`) so n8n also parses `body` as expected.
- **Version notes:** Node typeVersion `1.1` is older but fine; behavior differs slightly from newer webhook node versions regarding response configuration and rawBody handling.

---

### Block 2 — AI Image Generation & Base64 Conversion
**Overview:** Generates a LinkedIn-friendly illustration based on the quote, then converts the produced binary image into a base64 string for transport back to the website.  
**Nodes involved:** `Generate post image`, `Convert binary to base64`

#### Node: Generate post image
- **Type / Role:** `Google Gemini (Image)` via LangChain node — generates an image from a prompt.
- **Configuration choices:**
  - **Resource:** `image`
  - **Model:** `gemini-2.5-flash-image` (listed as “Nano Banana” in cached name)
  - **Prompt:** A detailed design brief:
    - Minimal but “visually fuller” concept illustration
    - Unique background color per output (rich professional tones)
    - Quote text overlay in **Poppins font only**
    - Flat design, polished, LinkedIn business aesthetic
    - Uses: `{{ $json.body.quote }}`
- **Credentials:** Google Gemini/PaLM API (`Google Gemini(PaLM) Api account`)
- **Connections:**
  - Output → `Convert binary to base64`
- **Edge cases / failures:**
  - Model availability: image model IDs can change; ensure the selected model exists in your Gemini account/region.
  - Font requirement (“Poppins”) is best-effort; image models may not reliably obey exact typography constraints.
  - Output is typically **binary**; downstream conversion assumes the image is indeed in binary output.
  - API limits/timeouts on image generation.

#### Node: Convert binary to base64
- **Type / Role:** `Extract From File` — converts binary to a JSON property (base64 string).
- **Configuration choices:**
  - **Operation:** `binaryToProperty`
  - **Destination key:** `image` (stores base64 string as `$json.image`)
- **Connections:**
  - Output → `Combine the branches` (as Input 1 / index 0)
- **Edge cases / failures:**
  - If the previous node doesn’t output a binary file (or uses a different binary property name than expected), conversion fails.
  - Large images can create large base64 payloads; watch webhook response size limits and browser memory usage.

---

### Block 3 — AI Caption Generation (Gemini Chat via Agent)
**Overview:** Produces a concise, professional LinkedIn caption from the quote using a Gemini chat model.  
**Nodes involved:** `Google Gemini Chat Model`, `Generate post caption`

#### Node: Google Gemini Chat Model
- **Type / Role:** LangChain Chat Model wrapper for Gemini — provides the LLM backend to the agent.
- **Configuration choices:**
  - Defaults (no special options set)
- **Credentials:** Google Gemini/PaLM API (`Google Gemini(PaLM) Api account`)
- **Connections:**
  - `ai_languageModel` output → `Generate post caption` as its language model input.
- **Edge cases / failures:**
  - Credential misconfiguration or API not enabled.
  - Model quota/rate limits.

#### Node: Generate post caption
- **Type / Role:** `LangChain Agent` — executes a single instruction (“define prompt”) using the connected chat model.
- **Configuration choices:**
  - **PromptType:** `define`
  - **Text prompt:** Generates a LinkedIn caption (120–180 words) with strong constraints:
    - educative/informative/engaging
    - no em dashes
    - no jargon/fluff
    - short paragraphs
    - reflective closing line without multiple questions
    - **Output only the final caption**
  - Uses: `{{ $json.body.quote }}`
- **Connections:**
  - Output → `Combine the branches` (as Input 2 / index 1)
- **Edge cases / failures:**
  - If the agent expects tools or memory configuration (not present here), it still works because it’s effectively a single-step generation.
  - Word-count constraints may not always be perfectly met; consider adding a post-check (IF + re-prompt) if strict compliance is required.
- **Version notes:** Node typeVersion `3.1` indicates newer agent behavior; ensure your n8n instance includes the `@n8n/n8n-nodes-langchain` package.

---

### Block 4 — Merge Results, Shape Response, Return to Website
**Overview:** Combines the caption and image into a single payload and returns it to the website webhook caller.  
**Nodes involved:** `Combine the branches`, `Set prompt and image`, `Respond to Webhook2`

#### Node: Combine the branches
- **Type / Role:** `Merge` — combines data from image branch and caption branch into one item.
- **Configuration choices:**
  - **Mode:** `Combine`
  - **Combine by:** `Position` (item 0 from each input combined)
- **Connections:**
  - Output → `Set prompt and image`
- **Edge cases / failures:**
  - If one branch returns no items (e.g., an error or empty output), merge may produce empty output or mismatch.
  - If branches return multiple items, position-based combining can pair unintended items (here it’s expected to be 1 item each).

#### Node: Set prompt and image
- **Type / Role:** `Set` — normalizes output fields for the website response.
- **Configuration choices:**
  - Creates/overwrites:
    - `image` = `{{ $json.image }}`
    - `output` = `{{ $json.output }}`
  - This implies:
    - base64 image is expected in `$json.image`
    - caption is expected in `$json.output` (LangChain agent commonly outputs `output`)
- **Connections:**
  - Output → `Respond to Webhook2`
- **Edge cases / failures:**
  - If the caption node outputs a different field name (e.g., `text`), `output` may be empty.
  - If the image conversion node stored base64 under a different property, `image` will be empty.

#### Node: Respond to Webhook2
- **Type / Role:** `Respond to Webhook` — returns the JSON payload to the website that initiated `Webhook - Receive Form`.
- **Configuration choices:**
  - **Respond with:** JSON
  - **Headers:**
    - `Content-Type: application/json`
    - `Access-Control-Allow-Origin: *` (CORS)
  - **Body:** `{{ JSON.stringify($json) }}`
    - Returns exactly the normalized object from the `Set` node.
- **Connections:** Final node for the “generate content” path.
- **Edge cases / failures:**
  - If response gets too large (base64 image), the browser/client may fail or time out.
  - CORS is permissive (`*`); if you need cookies/auth, you must tighten and configure accordingly.

---

### Block 5 — Publish to LinkedIn (Second Webhook)
**Overview:** Accepts a POST request (likely from the website after user edits) and creates a LinkedIn post via OAuth credentials, then responds with success.  
**Nodes involved:** `Webhook-LinkedIn-post`, `Create Linkedin post`, `Respond to Webhook`

#### Node: Webhook-LinkedIn-post
- **Type / Role:** `Webhook` — entry point for “publish now” action.
- **Configuration choices:**
  - **Path:** `d01e0a95-5a60-4c9f-9f4e-3e6ef694f402` (a UUID-like path)
  - **Method:** `POST`
  - **Response mode:** `Response Node` (explicit response is sent by `Respond to Webhook`)
- **Connections:**
  - Output → `Create Linkedin post`
- **Edge cases / failures:**
  - Payload structure must match what the LinkedIn node expects (text/media/org identifier).
  - If called from a browser, CORS preflight (OPTIONS) may be needed; webhook is configured for POST, not explicitly for OPTIONS.

#### Node: Create Linkedin post
- **Type / Role:** `LinkedIn` — creates a LinkedIn post.
- **Configuration choices:**
  - **Post as:** `organization`
  - **Additional fields:** none configured in the workflow
  - Note: The JSON does not show explicit post text/media parameters; in n8n UI this node typically requires fields such as author/organization URN, text, media, etc. They may be set implicitly, mapped via UI fields not visible here, or the node may currently be incomplete.
- **Credentials:** LinkedIn OAuth2 (`LinkedIn account`)
- **Connections:**
  - Output → `Respond to Webhook`
- **Edge cases / failures:**
  - Most common: OAuth scope issues, expired tokens, missing organization permissions.
  - Posting as organization requires the authenticated user to have admin/editor rights and correct scopes.
  - If attempting to post an image: LinkedIn often requires a multi-step upload process; ensure this node supports your intended media mode and the input payload includes required fields.

#### Node: Respond to Webhook
- **Type / Role:** `Respond to Webhook` — returns a fixed success message after posting.
- **Configuration choices:**
  - **Status:** 200
  - **Headers:**
    - `Access-Control-Allow-Origin: *`
    - `Access-Control-Allow-Methods: POST, OPTIONS`
    - `Access-Control-Allow-Headers: Content-Type`
    - `Content-Type: application/json`
  - **Body:** static JSON:
    - `{"success": true, "message": "Post received and posted to LinkedIn"}`
- **Connections:** Final node for publish path.
- **Edge cases / failures:**
  - This node always returns success if reached; consider branching on LinkedIn node errors and returning meaningful failure details to the client.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook - Receive Form | Webhook | Receives quote from website form; starts AI generation | — | Generate post image; Generate post caption | ## Upload your quote and generate a LinkedIn AI post design with captions  /  ## Watch this tutorial  /  @[youtube](T3zXQvoFB6A)  /  ## How it works  /  - Upload your quote on the website  /  - The webhook reads the data and generates an image and caption using Gemini AI  /  - Respond to webhook sends the response to the website, and it's displayed on the page  /  - Edit caption to taste and post to LinkedIn  /  ## Setup  /  1. Set up HTML website - will attach below  /  2. Replace the 2 webhooks in the HTML  /  3. Add [Gemini](https://aistudio.google.com/) credential  /  4. Add [LinkedIn](https://developer.linkedin.com/) credential  /  ## [Click to get the HTML website](https://drive.google.com/file/d/1JvViq-oNAdZosNOr7S5PfrxgbYcpK1Te/view)  /  ## 1. Retrieve the data from the website, generate a post image and caption, then send back the results to display on the website |
| Generate post image | Google Gemini (Image) | Generates post illustration from quote | Webhook - Receive Form | Convert binary to base64 | ## 1. Retrieve the data from the website, generate a post image and caption, then send back the results to display on the website |
| Convert binary to base64 | Extract From File | Converts generated image binary to base64 property | Generate post image | Combine the branches | ## 1. Retrieve the data from the website, generate a post image and caption, then send back the results to display on the website |
| Google Gemini Chat Model | Gemini Chat Model (LangChain) | Provides LLM backend for caption generation | — | Generate post caption (ai_languageModel) | ## 1. Retrieve the data from the website, generate a post image and caption, then send back the results to display on the website |
| Generate post caption | LangChain Agent | Generates LinkedIn caption from quote | Webhook - Receive Form; Google Gemini Chat Model | Combine the branches | ## 1. Retrieve the data from the website, generate a post image and caption, then send back the results to display on the website |
| Combine the branches | Merge | Combines caption + base64 image into one item | Convert binary to base64; Generate post caption | Set prompt and image | ## 1. Retrieve the data from the website, generate a post image and caption, then send back the results to display on the website |
| Set prompt and image | Set | Normalizes response fields (image, output) | Combine the branches | Respond to Webhook2 | ## 1. Retrieve the data from the website, generate a post image and caption, then send back the results to display on the website |
| Respond to Webhook2 | Respond to Webhook | Sends generated image+caption JSON back to website | Set prompt and image | — | ## 1. Retrieve the data from the website, generate a post image and caption, then send back the results to display on the website |
| Webhook-LinkedIn-post | Webhook | Receives “publish” request from website | — | Create Linkedin post | ## 2. Webhook reads the data and posts to LinkedIn |
| Create Linkedin post | LinkedIn | Publishes post to LinkedIn as organization | Webhook-LinkedIn-post | Respond to Webhook | ## 2. Webhook reads the data and posts to LinkedIn |
| Respond to Webhook | Respond to Webhook | Returns success response to publishing caller | Create Linkedin post | — | ## 2. Webhook reads the data and posts to LinkedIn |

---

## 4. Reproducing the Workflow from Scratch

1. **Create workflow** in n8n: name it “Create LinkedIn image posts with captions using Google Gemini and LinkedIn”.

2. **Add Webhook (Form intake)**
   - Node: **Webhook**
   - Name: `Webhook - Receive Form`
   - Method: `POST`
   - Path: `Linkedin-update`
   - Options: enable **Raw Body**
   - Response mode: **Last Node**
   - Expect input JSON containing: `body.quote` (e.g., `{ "quote": "..." }` coming from a website form).

3. **Add Gemini Image generation**
   - Node: **Google Gemini** (LangChain image node)
   - Name: `Generate post image`
   - Resource: `image`
   - Model: `gemini-2.5-flash-image`
   - Prompt: reference the webhook quote using an expression like `{{ $json.body.quote }}`
   - **Credentials:** create/connect a Google Gemini/PaLM API credential (from https://aistudio.google.com/).

4. **Add Base64 conversion**
   - Node: **Extract From File**
   - Name: `Convert binary to base64`
   - Operation: **Binary to Property**
   - Destination key: `image`
   - Connect: `Generate post image` → `Convert binary to base64`.

5. **Add Gemini Chat Model**
   - Node: **Google Gemini Chat Model**
   - Name: `Google Gemini Chat Model`
   - Credentials: same Gemini credential as above.

6. **Add Caption generation Agent**
   - Node: **AI Agent** (LangChain agent)
   - Name: `Generate post caption`
   - Prompt type: **Define**
   - Prompt: include `{{ $json.body.quote }}` and your caption constraints (120–180 words, no em dashes, etc.).
   - Connect the model: `Google Gemini Chat Model` → `Generate post caption` using the **ai_languageModel** connection.

7. **Connect intake webhook to both branches**
   - Connect: `Webhook - Receive Form` → `Generate post image`
   - Connect: `Webhook - Receive Form` → `Generate post caption`

8. **Merge image + caption**
   - Node: **Merge**
   - Name: `Combine the branches`
   - Mode: **Combine**
   - Combine by: **Position**
   - Connect:
     - `Convert binary to base64` → `Combine the branches` (Input 1)
     - `Generate post caption` → `Combine the branches` (Input 2)

9. **Shape the response payload**
   - Node: **Set**
   - Name: `Set prompt and image`
   - Add fields:
     - `image` = `{{ $json.image }}`
     - `output` = `{{ $json.output }}`
   - Connect: `Combine the branches` → `Set prompt and image`

10. **Return results to the website**
   - Node: **Respond to Webhook**
   - Name: `Respond to Webhook2`
   - Respond with: **JSON**
   - Headers:
     - `Content-Type: application/json`
     - `Access-Control-Allow-Origin: *`
   - Body: `{{ JSON.stringify($json) }}`
   - Connect: `Set prompt and image` → `Respond to Webhook2`

11. **Add Webhook (Publish to LinkedIn)**
   - Node: **Webhook**
   - Name: `Webhook-LinkedIn-post`
   - Method: `POST`
   - Path: use a unique path (the example uses a UUID-like string)
   - Response mode: **Response Node**
   - This webhook should receive the final caption and (optionally) image/media reference, depending on how you implement LinkedIn posting from your site.

12. **Add LinkedIn node**
   - Node: **LinkedIn**
   - Name: `Create Linkedin post`
   - Operation: create post (as supported by your n8n version)
   - Post as: **organization**
   - **Credentials:** create/connect LinkedIn OAuth2 credentials from https://developer.linkedin.com/
     - Ensure required scopes for posting (and organization posting if applicable).
     - Ensure the authenticated user has permission to post as the organization.
   - Map incoming webhook fields (caption text, and media if used) into the LinkedIn node parameters in the UI.

13. **Add publish response**
   - Node: **Respond to Webhook**
   - Name: `Respond to Webhook`
   - Status: `200`
   - Headers:
     - `Access-Control-Allow-Origin: *`
     - `Access-Control-Allow-Methods: POST, OPTIONS`
     - `Access-Control-Allow-Headers: Content-Type`
     - `Content-Type: application/json`
   - Body (static):
     - `{ "success": true, "message": "Post received and posted to LinkedIn" }`

14. **Connect publish chain**
   - `Webhook-LinkedIn-post` → `Create Linkedin post` → `Respond to Webhook`

15. **Website integration**
   - Use the provided HTML site (see link below).
   - Replace both webhook URLs in the HTML with your n8n production webhook URLs:
     - One for `/webhook/Linkedin-update`
     - One for `/webhook/<publish-path>`

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Watch this video | https://www.youtube.com/watch?v=T3zXQvoFB6A |
| HTML website download | https://drive.google.com/file/d/1JvViq-oNAdZosNOr7S5PfrxgbYcpK1Te/view |
| Gemini credential setup | https://aistudio.google.com/ |
| LinkedIn developer credentials | https://developer.linkedin.com/ |
| Workflow behavior summary: upload quote → generate image+caption → preview/edit → post to LinkedIn | Sticky note “How it works” section in the workflow |


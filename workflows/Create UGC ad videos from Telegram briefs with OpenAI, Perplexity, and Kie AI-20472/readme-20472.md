Create UGC ad videos from Telegram briefs with OpenAI, Perplexity, and Kie AI

https://n8nworkflows.xyz/workflows/create-ugc-ad-videos-from-telegram-briefs-with-openai--perplexity--and-kie-ai-20472


# Create UGC ad videos from Telegram briefs with OpenAI, Perplexity, and Kie AI

### 1. Workflow Overview

This workflow automates the generation of User-Generated Content (UGC) ad videos using a Telegram chatbot interface combined with advanced AI models. Users submit product details, target orientation, concept descriptions, and a product image via a Telegram form. The system then processes the inputs, researches the target audience, builds a multi-shot video storyboard, generates a foundational first-frame image, renders an AI-powered storyboard video via Kie.ai, and delivers the final MP4 video directly back to the user on Telegram.

The workflow logic is categorized into six sequential blocks:
- **1.1 Input Reception & Upload:** Triggers upon receiving a Telegram message, presents a multi-field data collection form, uploads the attached product photo to `imgbb` to generate a public URL, and confirms receipt to the user.
- **1.2 Research & Image Analysis:** Standardizes the video aspect ratio, analyzes the uploaded product image using OpenAI (GPT-4o Vision), and researches the product and target audience using Perplexity.
- **1.3 Storyboard Generation:** Combines user inputs, image analysis, and market research to prompt an AI Agent powered by OpenAI (GPT-5.1) and a Structured Output Parser, producing a multi-shot storyboard and a first-frame image prompt, which is then cleaned by a custom JavaScript code node.
- **1.4 First-Frame Image Generation:** Requests an opening image from Kie.ai (Nano Banana Pro) based on the storyboard's first-frame prompt, polls the generation status every 2 minutes, handles errors, and confirms completion.
- **1.5 Video Rendering & Polling:** Formats the compiled shot list and opening image URL into a Kie.ai Sora 2 Pro Storyboard payload, initiates video rendering, polls the job status every 10 minutes, handles potential errors, and extracts the final download link.
- **1.6 Delivery:** Downloads the rendered MP4 file from Kie.ai and delivers both the video file and a direct clickable URL to the user via Telegram.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Upload
- **Overview:** Captures the initial user interaction on Telegram, serves an interactive form to gather project details and binary image files, uploads the image to a public repository (`imgbb`), and initiates processing delays.
- **Nodes Involved:** `Telegram'ı Çalıştır`, `Form Bilgisi Topla`, `Fotoğrafı Linke Çevir`, `Bekle 5 sn`, `Form Özeti`, `Bekleyiniz`.
- **Node Details:**
  - `Telegram'ı Çalıştır` (Trigger / Telegram Trigger): Listens for incoming chat updates. Input: None. Output: Raw Telegram message payload. Version: 1.2. Edge cases: Webhook desync or missing bot token permissions.
  - `Form Bilgisi Topla` (Action / Telegram): Sends a custom interactive form (`sendAndWait`) collecting brand name, product description, image, duration, scene count, orientation, concept prompt, and language. Input: Telegram chat ID. Output: Form responses including binary file data. Version: 1.2. Edge cases: Invalid file formats or timeouts.
  - `Fotoğrafı Linke Çevir` (Action / HTTP Request): Uploads the binary photo to `imgbb.com` via multipart/form-data. Input: Binary image from form. Output: Public image URL inside JSON response. Version: 4.3. Requires API key parameter. Edge cases: `imgbb` rate limits, expired keys, or unsupported file formats.
  - `Bekle 5 sn` (Action / Wait): Pauses execution for 5 seconds to ensure rate limit compliance. Input: HTTP response. Output: Unchanged item data. Version: 1.1.
  - `Form Özeti` (Action / Telegram): Sends a Markdown-formatted summary of submitted inputs back to the user. Input: Form data. Output: Telegram message confirmation. Version: 1.2.
  - `Bekleyiniz` (Action / Telegram): Sends an estimated wait-time notice (12–30 minutes) to the user. Input: Telegram metadata. Output: Status notification. Version: 1.2.

#### 2.2 Research & Image Analysis
- **Overview:** Normalizes user-selected video orientation into standard aspect ratios, runs multimodal image analysis using GPT-4o, and executes market and audience research via Perplexity.
- **Nodes Involved:** `Aspect Ratio`, `Ürün Görselini İncele`, `Gemini - Ürün İncele` (Disabled), `Ürünü Araştır`.
- **Node Details:**
  - `Aspect Ratio` (Action / Code): Maps form orientation selections (`portrait`, `landscape`, `dikey`, `yatay`) to standard aspect ratio strings (`9:16` or `16:9`). Input: Form data. Output: Enriched item with `aspect_ratio`. Version: 2. Edge cases: Unhandled text strings default to `9:16`.
  - `Ürün Görselini İncele` (Action / OpenAI LangChain): Uses GPT-4o to analyze the public image URL. Input: Public image URL. Output: Textual description of the product image. Version: 2. Requires OpenAI API credentials. Edge cases: Inaccessible image URLs or API rate limits.
  - `Gemini - Ürün İncele` (Action / Google Gemini LangChain): Alternative multimodal analyzer. *Disabled by default.* Version: 1.
  - `Ürünü Araştır` (Action / Perplexity): Queries Perplexity AI with product details, brand context, and visual descriptions to generate target audience insights. Input: Form fields and image text analysis. Output: Markdown research summary (`$json.message`). Version: 1. Requires Perplexity API credentials. Edge cases: API timeouts or strict content filters.

#### 2.3 Storyboard Generation
- **Overview:** Coordinates an LLM agent to synthesize research and user constraints into a structured multi-shot storyboard and cleans the output via custom JavaScript.
- **Nodes Involved:** `AI Agent`, `OpenAI Chat Model`, `Structured Output Parser`, `Shots Builder`, `Senaryo Tamamlandı`.
- **Node Details:**
  - `AI Agent` (Action / LangChain Agent): Generates UGC ad prompts, shot durations, and a first-frame image prompt based on a detailed system prompt and user variables. Input: Form fields, image analysis, and Perplexity research. Output: Structured JSON payload. Version: 2.2.
  - `OpenAI Chat Model` (Action / LangChain Model): Configured with `gpt-5.1` to power the AI Agent and Structured Output Parser. Version: 1.2. Requires OpenAI credentials.
  - `Structured Output Parser` (Action / LangChain Output Parser): Enforces a strict JSON schema containing `shots` arrays (Scene description and duration) and `first_scene_img_prompt`. Version: 1.3. Edge cases: LLM output deviations trigger auto-fixing routines.
  - `Shots Builder` (Action / Code): Sanitizes LLM string outputs (escaping quotes, normalizing whitespace) and builds a clean array of shots and safe image prompts. Input: LLM JSON output. Output: Normalized JSON payload containing `shots` and `first_scene_img_prompt_safe`. Version: 2.
  - `Senaryo Tamamlandı` (Action / Telegram): Notifies the user that image analysis, research, and scriptwriting are finished. Input: Cleaned storyboard data. Output: Telegram message. Version: 1.2.

#### 2.4 First-Frame Image Generation
- **Overview:** Triggers an external image generation job (Nano Banana Pro) to create the opening frame, polls the status periodically, and handles processing errors.
- **Nodes Involved:** `Nano Banana Pro`, `Nano Banana` (Disabled), `Bekle 2dk`, `Görseli Al`, `Görsel Kontrol`, `Görsel Üretildi`, `Görsel Error`.
- **Node Details:**
  - `Nano Banana Pro` (Action / HTTP Request): Sends a POST request to Kie.ai (`/api/v1/jobs/createTask`) for model `nano-banana-pro`. Input: Safe first-frame prompt, aspect ratio, and source image URL. Output: Task creation ID (`taskId`). Version: 4.3. Requires HTTP Header Auth (`Authorization: Bearer <API_KEY>`). Edge cases: Invalid API key or credit exhaustion.
  - `Nano Banana` (Action / HTTP Request): Alternative legacy image model. *Disabled by default.* Version: 4.3.
  - `Bekle 2dk` (Action / Wait): Pauses execution for 2 minutes to allow image rendering. Version: 1.1.
  - `Görseli Al` (Action / HTTP Request): Queries Kie.ai (`/api/v1/jobs/recordInfo`) using the generated `taskId` to check task status. Version: 4.3.
  - `Görsel Kontrol` (Action / Switch): Evaluates job states (`ing` for processing, `success` for completion, or fallback for errors). Input: Task status payload. Version: 3.3.
  - `Görsel Üretildi` (Action / Telegram): Confirms successful first-frame generation. Version: 1.2.
  - `Görsel Error` (Action / Telegram): Reports failure codes, messages, and credit refund notices to the user via Telegram. Version: 1.2.

#### 2.5 Video Rendering & Polling
- **Overview:** Formats all assets into a storyboard video payload, sends it to Kie.ai, polls the generation job every 10 minutes, and handles video rendering errors.
- **Nodes Involved:** `Formatlama`, `Video Uret`, `Bekle 10dk`, `Videoyu Al`, `Video Kontrol`, `Video Üretildi`, `Video Error`, `Link Formatlama`.
- **Node Details:**
  - `Formatlama` (Action / Code): Parses the generated first-frame image URL and maps it alongside total duration, aspect ratio, and shots into the Kie.ai Sora 2 Pro payload. Input: Image retrieval results and form parameters. Output: JSON payload for `Video Uret`. Version: 2.
  - `Video Uret` (Action / HTTP Request): Submits the Sora 2 Pro Storyboard task (`sora-2-pro-storyboard`) to Kie.ai (`/api/v1/jobs/createTask`). Input: Formatted payload. Output: Video generation `taskId`. Version: 4.2. Requires HTTP Header Auth.
  - `Bekle 10dk` (Action / Wait): Pauses execution for 10 minutes between video polling requests. Version: 1.1.
  - `Videoyu Al` (Action / HTTP Request): Queries Kie.ai record info using the video `taskId`. Version: 4.2.
  - `Video Kontrol` (Action / Switch): Routes execution based on video processing state (`ing`, `success`, or error fallback). Version: 3.3.
  - `Video Üretildi` (Action / Telegram): Sends a success notice for video rendering. Version: 1.2.
  - `Video Error` (Action / Telegram): Reports video generation errors and credit refund statuses to Telegram. Version: 1.2.
  - `Link Formatlama` (Action / Code): Extracts and cleans the final MP4 download URL from the Kie.ai response object. Input: Successful video API record payload. Output: Cleaned `{ url: "<mp4 link>" }`. Version: 2.

#### 2.6 Delivery
- **Overview:** Downloads the rendered MP4 file from the remote URL and sends both the binary file and a clickable link to the user on Telegram.
- **Nodes Involved:** `Videoyu İndir`, `Videoyu Gönder`, `Video Linki Gönder`.
- **Node Details:**
  - `Videoyu İndir` (Action / HTTP Request): Downloads the MP4 file from the extracted URL into n8n binary memory. Input: Cleaned MP4 URL. Output: Binary file data. Version: 4.3. Edge cases: Broken URLs or network timeouts.
  - `Videoyu Gönder` (Action / Telegram): Uploads and sends the binary MP4 video file to the user's Telegram chat (`sendVideo`). Input: Binary file and chat ID. Output: Telegram message confirmation. Version: 1.2. Edge cases: Telegram file size limits (typically 50MB for bots).
  - `Video Linki Gönder` (Action / Telegram): Sends a Markdown-formatted clickable hyperlink pointing directly to the MP4 file. Input: Cleaned URL. Output: Telegram message. Version: 1.2.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Telegram'ı Çalıştır` | Telegram Trigger | Triggers workflow on incoming message | None | `Form Bilgisi Topla` | ## 1. Collect product info<br>Telegram form gathers the brief, uploads the photo to imgbb and confirms the details. |
| `Form Bilgisi Topla` | Telegram | Displays form to collect inputs and photo | `Telegram'ı Çalıştır` | `Fotoğrafı Linke Çevir` | ## 1. Collect product info<br>Telegram form gathers the brief, uploads the photo to imgbb and confirms the details. |
| `Fotoğrafı Linke Çevir` | HTTP Request | Uploads product photo to imgbb | `Form Bilgisi Topla` | `Bekle 5 sn` | ## 1. Collect product info<br>Telegram form gathers the brief, uploads the photo to imgbb and confirms the details. |
| `Bekle 5 sn` | Wait | Rate-limit delay | `Fotoğrafı Linke Çevir` | `Form Özeti` | ## 1. Collect product info<br>Telegram form gathers the brief, uploads the photo to imgbb and confirms the details. |
| `Form Özeti` | Telegram | Sends form summary to Telegram chat | `Bekle 5 sn` | `Bekleyiniz` | ## 1. Collect product info<br>Telegram form gathers the brief, uploads the photo to imgbb and confirms the details. |
| `Bekleyiniz` | Telegram | Sends processing time estimate | `Form Özeti` | `Aspect Ratio` | ## 1. Collect product info<br>Telegram form gathers the brief, uploads the photo to imgbb and confirms the details. |
| `Aspect Ratio` | Code | Maps orientation string to aspect ratio | `Bekleyiniz` | `Ürün Görselini İncele` | ## 2. Research the product<br>GPT-4o describes the product photo; Perplexity researches the product and audience. |
| `Ürün Görselini İncele` | OpenAI | Multimodal image description (GPT-4o) | `Aspect Ratio` | `Ürünü Araştır` | ## 2. Research the product<br>GPT-4o describes the product photo; Perplexity researches the product and audience. |
| `Gemini - Ürün İncele` | Google Gemini | Alternative image analysis (Disabled) | None | None | ## 2. Research the product<br>GPT-4o describes the product photo; Perplexity researches the product and audience. |
| `Ürünü Araştır` | Perplexity | Researches product and target audience | `Ürün Görselini İncele` | `AI Agent` | ## 2. Research the product<br>GPT-4o describes the product photo; Perplexity researches the product and audience. |
| `AI Agent` | LangChain Agent | Generates storyboard structure | `Ürünü Araştır`, `Form Bilgisi Topla`, `Ürün Görselini İncele` | `Shots Builder` | ## 3. Write the storyboard<br>AI agent writes the shots and first-frame prompt; Shots Builder cleans the output. |
| `OpenAI Chat Model` | OpenAI Chat Model | LLM backend for AI Agent & Parser | None | `AI Agent`, `Structured Output Parser` | ## 3. Write the storyboard<br>AI agent writes the shots and first-frame prompt; Shots Builder cleans the output. |
| `Structured Output Parser` | Structured Output Parser | Enforces JSON output schema | None | `AI Agent` | ## 3. Write the storyboard<br>AI agent writes the shots and first-frame prompt; Shots Builder cleans the output. |
| `Shots Builder` | Code | Sanitizes storyboard and image prompt | `AI Agent` | `Senaryo Tamamlandı` | ## 3. Write the storyboard<br>AI agent writes the shots and first-frame prompt; Shots Builder cleans the output. |
| `Senaryo Tamamlandı` | Telegram | Notifies user script is complete | `Shots Builder` | `Nano Banana Pro` | ## 3. Write the storyboard<br>AI agent writes the shots and first-frame prompt; Shots Builder cleans the output. |
| `Nano Banana Pro` | HTTP Request | Requests opening image generation | `Senaryo Tamamlandı` | `Bekle 2dk` | ## 4. Generate the first frame<br>Nano Banana Pro creates the opening image; status is checked every 2 minutes. |
| `Nano Banana` | HTTP Request | Alternative image model (Disabled) | None | None | ## 4. Generate the first frame<br>Nano Banana Pro creates the opening image; status is checked every 2 minutes. |
| `Bekle 2dk` | Wait | Wait delay for image generation | `Nano Banana Pro` | `Görseli Al` | ## 4. Generate the first frame<br>Nano Banana Pro creates the opening image; status is checked every 2 minutes. |
| `Görseli Al` | HTTP Request | Polls image generation task status | `Bekle 2dk` | `Görsel Kontrol` | ## 4. Generate the first frame<br>Nano Banana Pro creates the opening image; status is checked every 2 minutes. |
| `Görsel Kontrol` | Switch | Routes based on image job status | `Görseli Al` | `Bekle 2dk`, `Görsel Üretildi`, `Görsel Error` | ## 4. Generate the first frame<br>Nano Banana Pro creates the opening image; status is checked every 2 minutes. |
| `Görsel Üretildi` | Telegram | Confirms image generation success | `Görsel Kontrol` | `Formatlama` | ## 4. Generate the first frame<br>Nano Banana Pro creates the opening image; status is checked every 2 minutes. |
| `Görsel Error` | Telegram | Reports image generation failure | `Görsel Kontrol` | None | ## 4. Generate the first frame<br>Nano Banana Pro creates the opening image; status is checked every 2 minutes. |
| `Formatlama` | Code | Prepares Sora 2 Pro Storyboard payload | `Görsel Üretildi` | `Video Uret` | ## 5. Generate the video<br>Builds the Sora 2 Pro Storyboard payload and polls the job every 10 minutes. |
| `Video Uret` | HTTP Request | Submits video creation task | `Formatlama` | `Bekle 10dk` | ## 5. Generate the video<br>Builds the Sora 2 Pro Storyboard payload and polls the job every 10 minutes. |
| `Bekle 10dk` | Wait | Wait delay for video rendering | `Video Uret` | `Videoyu Al` | ## 5. Generate the video<br>Builds the Sora 2 Pro Storyboard payload and polls the job every 10 minutes. |
| `Videoyu Al` | HTTP Request | Polls video generation task status | `Bekle 10dk` | `Video Kontrol` | ## 5. Generate the video<br>Builds the Sora 2 Pro Storyboard payload and polls the job every 10 minutes. |
| `Video Kontrol` | Switch | Routes based on video job status | `Videoyu Al` | `Bekle 10dk`, `Link Formatlama`, `Video Error` | ## 5. Generate the video<br>Builds the Sora 2 Pro Storyboard payload and polls the job every 10 minutes. |
| `Video Üretildi` | Telegram | Confirms video rendering success | `Link Formatlama` | `Videoyu İndir` | ## 5. Generate the video<br>Builds the Sora 2 Pro Storyboard payload and polls the job every 10 minutes. |
| `Video Error` | Telegram | Reports video rendering failure | `Video Kontrol` | None | ## 5. Generate the video<br>Builds the Sora 2 Pro Storyboard payload and polls the job every 10 minutes. |
| `Link Formatlama` | Code | Extracts final MP4 URL | `Video Kontrol` | `Video Üretildi`, `Video Linki Gönder` | ## 5. Generate the video<br>Builds the Sora 2 Pro Storyboard payload and polls the job every 10 minutes. |
| `Videoyu İndir` | HTTP Request | Downloads MP4 binary file | `Link Formatlama` | `Videoyu Gönder` | ## 6. Deliver the video<br>Downloads the result and sends the file and link on Telegram. |
| `Videoyu Gönder` | Telegram | Sends MP4 video file to Telegram | `Videoyu İndir` | `Video Linki Gönder` | ## 6. Deliver the video<br>Downloads the result and sends the file and link on Telegram. |
| `Video Linki Gönder` | Telegram | Sends clickable video link | `Videoyu Gönder`, `Link Formatlama` | None | ## 6. Deliver the video<br>Downloads the result and sends the file and link on Telegram. |

---

### 4. Reproducing the Workflow from Scratch

1. **Trigger Setup:**
   - Create a **Telegram Trigger** node named `Telegram'ı Çalıştır`. Configure credentials and set updates to listen for `message`.
2. **Form Collection & Image Upload:**
   - Add a **Telegram** node named `Form Bilgisi Topla`. Set operation to `sendAndWait` with custom form fields: Brand Name (text), Product Description (text), Product Photo (`file`, accepts images), Video Length (dropdown: 10, 15, 25), Scene Count (dropdown: 2 to 7), Orientation (dropdown: `portrait`, `landscape`), Concept Prompt (text), Spoken Language (text).
   - Add an **HTTP Request** node named `Fotoğrafı Linke Çevir`. Method: `POST`, URL: `https://api.imgbb.com/1/upload`, Body Content Type: `multipart/form-data`. Add body parameter `image` (Form Binary Data from form input) and query parameter `key` (imgbb API key).
   - Add a **Wait** node named `Bekle 5 sn` (default settings).
   - Add two **Telegram** nodes: `Form Özeti` and `Bekleyiniz`, configured with Markdown parse mode to send summary and waiting notices.
3. **Aspect Ratio & Analysis:**
   - Add a **Code** node named `Aspect Ratio` to normalize orientation inputs into `9:16` or `16:9`.
   - Add an **OpenAI** node named `Ürün Görselini İncele`. Resource: `Image`, Operation: `Analyze`, Model: `gpt-4o`. Feed the image URL from `Fotoğrafı Linke Çevir`.
   - Add a **Perplexity** node named `Ürünü Araştır`. Feed brand details, product text, and GPT-4o analysis results into the message body.
4. **Storyboard Generation:**
   - Add an **AI Agent** node named `AI Agent`. Configure prompt variables referencing previous node outputs. Attach an **OpenAI Chat Model** (`gpt-5.1`) and a **Structured Output Parser** configured with a JSON schema for `shots` and `first_scene_img_prompt`.
   - Add a **Code** node named `Shots Builder` using JavaScript to sanitize strings and output safe prompt variables.
   - Add a **Telegram** node named `Senaryo Tamamlandı` to notify the user.
5. **First-Frame Image Generation:**
   - Add an **HTTP Request** node named `Nano Banana Pro`. Method: `POST`, URL: `https://api.kie.ai/api/v1/jobs/createTask`. Configure Generic HTTP Header Auth (`Authorization: Bearer <API_KEY>`). Body JSON payload must specify model `nano-banana-pro`, safe prompt, aspect ratio, and source image URL.
   - Add a **Wait** node named `Bekle 2dk` (Amount: 2, Unit: minutes).
   - Add an **HTTP Request** node named `Görseli Al`. Method: `GET`, URL: `https://api.kie.ai/api/v1/jobs/recordInfo`, Query parameter `taskId` pointing to the task ID from `Nano Banana Pro`.
   - Add a **Switch** node named `Görsel Kontrol` with rules matching state `ing` (processing) looping back to `Bekle 2dk`, `success` proceeding forward, and fallback leading to `Görsel Error`.
   - Add **Telegram** nodes `Görsel Üretildi` and `Görsel Error`.
6. **Video Rendering & Polling:**
   - Add a **Code** node named `Formatlama` to construct the Sora 2 Pro Storyboard payload (`sora-2-pro-storyboard`, aspect ratio, total duration, shots, and first-frame image URL).
   - Add an **HTTP Request** node named `Video Uret`. Method: `POST`, URL: `https://api.kie.ai/api/v1/jobs/createTask`, using HTTP Header Auth.
   - Add a **Wait** node named `Bekle 10dk` (Amount: 10, Unit: minutes).
   - Add an **HTTP Request** node named `Videoyu Al`. Method: `GET`, URL: `https://api.kie.ai/api/v1/jobs/recordInfo`, Query parameter `taskId`.
   - Add a **Switch** node named `Video Kontrol` routing `ing` back to `Bekle 10dk`, `success` forward, and errors to `Video Error`.
   - Add **Telegram** nodes `Video Üretildi` and `Video Error`.
   - Add a **Code** node named `Link Formatlama` to extract the final MP4 URL.
7. **Delivery:**
   - Add an **HTTP Request** node named `Videoyu İndir` to download the binary MP4 from the extracted URL.
   - Add **Telegram** nodes `Videoyu Gönder` (`sendVideo` operation with binary data) and `Video Linki Gönder` (Markdown clickable link).
8. **Connections:**
   - Connect all nodes sequentially according to the data flow described in the block analysis and summary table.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Turn a Telegram form into Sora 2 Pro storyboard UGC ads | Primary workflow objective and architecture overview |
| Telegram Bot Setup | Create a bot via [@BotFather](https://t.me/BotFather) and configure bot credentials |
| imgbb API Setup | Obtain free API keys via [imgbb Account About > API](https://imgbb.com/) |
| OpenAI API Setup | Add billing credit and API keys at [OpenAI Platform](https://platform.openai.com) |
| Perplexity API Setup | Add API billing credits and keys at [Perplexity Account API](https://www.perplexity.ai) |
| Kie.ai API Setup | Top up credits, obtain API key, and configure Header Auth (`Authorization: Bearer <API_KEY>`) via [Kie.ai](https://kie.ai?ref=c29fcd9c92ad897e8868df9182112560) |
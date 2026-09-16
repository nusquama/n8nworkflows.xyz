Generate quiet luxury lifestyle product images with OpenAI and Google Drive

https://n8nworkflows.xyz/workflows/generate-quiet-luxury-lifestyle-product-images-with-openai-and-google-drive-18417


# Generate quiet luxury lifestyle product images with OpenAI and Google Drive

### 1. Workflow Overview

The **AI Product Image Generator** workflow automates the creation of high-end, "Quiet Luxury" lifestyle product photography from user-submitted inputs. Target use cases include e-commerce visual content generation, brand marketing, and automated digital asset creation. 

The workflow is structured into the following logical blocks:
- **1.1 Input Reception & Staging:** Captures form submissions containing product imagery, scene briefs, and model gender preferences, then stages the raw file in Google Drive.
- **1.2 Visual Analysis & Planning:** Extracts binary data, performs visual analysis via OpenAI, and uses an AI Agent powered by structured output parsing to break down scene briefs into distinct creative variations.
- **1.3 Parallel Image Generation & Upload:** Fan-out architecture executing three independent generation branches. Each branch downloads the staged source image, refines prompts using dedicated OpenAI chat models, requests image edits from the OpenAI API (`gpt-image-2`), converts the base64 output responses to binary files, and uploads the results to Google Drive.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Staging
This block captures the initial user input and ensures the product image is stored and retrievable via a stable file reference in Google Drive.

- **When Form Submitted**
  - **Type & Role:** `n8n-nodes-base.formTrigger` — Webhook-based trigger node that creates an interactive public form.
  - **Configuration:** Configured with three form fields: `Scene Brief` (Text), `Image` (File), and `female/Man` (Text).
  - **Key Expressions:** None.
  - **Connections:** Output connects to `Upload file3`.
  - **Edge Cases:** Missing file uploads or unpopulated text fields will fail downstream execution.

- **Upload file3**
  - **Type & Role:** `n8n-nodes-base.googleDrive` — Uploads the form-submitted binary image to Google Drive.
  - **Configuration:** Operation: `upload`, Drive ID: `My Drive`, Folder: Root (`root`), Input Data Field Name: `Image`, File Name: `={{ $json.Image[0].filename }}`.
  - **Key Expressions:** `={{ $json.Image[0].filename }}`
  - **Connections:** Input from `When Form Submitted`, output to `Download file`.
  - **Edge Cases:** Google Drive storage limits, authentication token expiration.

- **Download file**
  - **Type & Role:** `n8n-nodes-base.googleDrive` — Downloads the newly uploaded file back from Google Drive to establish a consistent file reference.
  - **Configuration:** Operation: `download`, File ID: `={{ $json.id }}`.
  - **Key Expressions:** `={{ $json.id }}`
  - **Connections:** Input from `Upload file3`, output to `Extract Data from File`.
  - **Edge Cases:** API rate limits or latency preventing immediate availability of the file ID.

---

#### 2.2 Visual Analysis & Planning
This block processes the downloaded file, analyzes its visual characteristics and brand aesthetic using OpenAI Vision, and structures the user's scene brief into actionable variation directions.

- **Extract Data from File**
  - **Type & Role:** `n8n-nodes-base.extractFromFile` — Extracts binary data from the downloaded file container.
  - **Configuration:** Operation: `binaryToProperty`.
  - **Connections:** Input from `Download file`, output to `Convert Data to File`.

- **Convert Data to File**
  - **Type & Role:** `n8n-nodes-base.convertToFile` — Prepares extracted binary data into a format suitable for the OpenAI vision analyzer.
  - **Configuration:** Operation: `toBinary`, Source Property: `data`.
  - **Connections:** Input from `Extract Data from File`, output to `OpenAI Image Analysis`.

- **OpenAI Image Analysis**
  - **Type & Role:** `@n8n/n8n-nodes-langchain.openAi` — Performs multimodal image analysis to preserve brand vibe.
  - **Configuration:** Model: `gpt-4o`, Resource: `image`, Input Type: `base64`, Operation: `analyze`.
  - **Key Expressions:** Text prompt: `"Analyse the image. Understand the vibe of the brand. Make sure to keep the vibe."`
  - **Connections:** Input from `Convert Data to File`, output to `AI Analysis Agent`.
  - **Edge Cases:** OpenAI service outages, strict prompt safety filters.

- **OpenAI GPT-4 Chat Model**
  - **Type & Role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — Language model provider backing the AI Agent.
  - **Configuration:** Model: `gpt-4.1-mini`.
  - **Connections:** Connected to `AI Analysis Agent` via `ai_languageModel`.

- **Parse Structured Output**
  - **Type & Role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — Enforces a strict JSON schema output format for the AI Agent.
  - **Configuration:** JSON Schema enforces an array of three distinct scene briefs under the `scenes` key.
  - **Connections:** Connected to `AI Analysis Agent` via `ai_outputParser`.

- **AI Analysis Agent**
  - **Type & Role:** `@n8n/n8n-nodes-langchain.agent` — LangChain agent orchestrating prompt parsing based on user input.
  - **Configuration:** Prompt Type: `define`, System Message injects the scene brief from the form trigger and requests JSON formatting.
  - **Key Expressions:** Text: `={{ $('When Form Submitted').item.json['Scene Brief'] }}`
  - **Connections:** Input from `OpenAI Image Analysis`, outputs in parallel to `Message a model1`, `OpenAI Message Model`, and `Send Message to OpenAI`.

---

#### 2.3 Parallel Image Generation & Upload
This block fans out into three parallel processing pipelines. Each pipeline downloads the source image, formats customized prompts using branch-specific AI models, calls OpenAI's image edits endpoint, converts the response to a binary file, and uploads the final asset to Google Drive.

##### Branch 1 (Primary / Detailed IMG)
- **Message a model1**
  - **Type & Role:** `@n8n/n8n-nodes-langchain.openAi` — Refines the user prompt for the first branch, incorporating model gender preferences and Quiet Luxury aesthetics.
  - **Configuration:** Model: `gpt-4.1`.
  - **Key Expressions:** Content: `Refine the User prompt: {{ $('When Form Submitted').item.json['Scene Brief'] }} ... Model: {{ $('When Form Submitted').item.json['female/Man'] }}`
  - **Connections:** Input from `AI Analysis Agent`, output to `Download file1`.

- **Download file1**
  - **Type & Role:** `n8n-nodes-base.googleDrive` — Downloads the reference source image from Google Drive for branch processing.
  - **Configuration:** Operation: `download`, File ID references `Download file`.
  - **Key Expressions:** `={{ $('Download file').first().json.id }}`
  - **Connections:** Input from `Message a model1`, output to `Post to OpenAI Image Edits`.

- **Post to OpenAI Image Edits**
  - **Type & Role:** `n8n-nodes-base.httpRequest` — Sends the source image and prompt to OpenAI's image edits endpoint.
  - **Configuration:** URL: `https://api.openai.com/v1/images/edits`, Method: `POST`, Content-Type: `multipart-form-data`, Authentication: Predefined OpenAI API credentials. Body parameters include prompt, image binary, model (`gpt-image-2`), size (`1024x1536`), and quality (`medium`).
  - **Connections:** Input from `Download file1`, output to `Convert JSON to File Format`.
  - **Edge Cases:** OpenAI image generation rate limits, timeouts on heavy rendering tasks.

- **Convert JSON to File Format**
  - **Type & Role:** `n8n-nodes-base.convertToFile` — Converts the base64 JSON response from OpenAI into binary file data.
  - **Configuration:** Operation: `toBinary`, Source Property: `data[0].b64_json`, Binary Property Name: `image`.
  - **Connections:** Input from `Post to OpenAI Image Edits`, output to `Upload File to Google Drive`.

- **Upload File to Google Drive**
  - **Type & Role:** `n8n-nodes-base.googleDrive` — Uploads the final generated lifestyle image.
  - **Configuration:** Operation: `upload`, Name: `Detailed IMG`, Drive ID: `My Drive`, Folder ID: Configured target folder placeholder.
  - **Connections:** Input from `Convert JSON to File Format`.

---

##### Branch 2 (Secondary / Product Images)
- **OpenAI Message Model**
  - **Type & Role:** `@n8n/n8n-nodes-langchain.openAi` — Refines prompt variation 2 from structured agent outputs.
  - **Configuration:** Model: `gpt-4o`.
  - **Key Expressions:** Content references `{{ $json.output.scenes[1] }}`.
  - **Connections:** Input from `AI Analysis Agent`, output to `Download file2`.

- **Download file2**
  - **Type & Role:** `n8n-nodes-base.googleDrive` — Downloads reference image for branch 2.
  - **Configuration:** Operation: `download`, File ID: `={{ $('Download file').first().json.id }}`.
  - **Connections:** Input from `OpenAI Message Model`, output to `Image Generation Model1`.

- **Image Generation Model1**
  - **Type & Role:** `n8n-nodes-base.httpRequest` — Calls OpenAI Image Edits endpoint (`gpt-image-2`) for branch 2.
  - **Configuration:** URL: `https://api.openai.com/v1/images/edits`, Method: `POST`, Content-Type: `multipart-form-data`. Uses refined text output from `OpenAI Message Model`.
  - **Connections:** Input from `Download file2`, output to `Convert to File1`.

- **Convert to File1**
  - **Type & Role:** `n8n-nodes-base.convertToFile` — Converts base64 generation response to binary.
  - **Configuration:** Operation: `toBinary`, Source Property: `data[0].b64_json`, Binary Property Name: `image`.
  - **Connections:** Input from `Image Generation Model1`, output to `Upload file1`.

- **Upload file1**
  - **Type & Role:** `n8n-nodes-base.googleDrive` — Uploads final branch 2 image.
  - **Configuration:** Operation: `upload`, Name: `Product images`, Drive ID: `My Drive`, Folder ID: Configured target folder placeholder.
  - **Connections:** Input from `Convert to File1`.

---

##### Branch 3 (Tertiary / Content)
- **Send Message to OpenAI**
  - **Type & Role:** `@n8n/n8n-nodes-langchain.openAi` — Refines prompt variation 3 from structured agent outputs.
  - **Configuration:** Model: `gpt-4o`.
  - **Key Expressions:** Content references travel lifestyle and Quiet Luxury styling guidelines.
  - **Connections:** Input from `AI Analysis Agent`, output to `Download file3`.

- **Download file3**
  - **Type & Role:** `n8n-nodes-base.googleDrive` — Downloads reference image for branch 3.
  - **Configuration:** Operation: `download`, File ID: `={{ $('Download file').first().json.id }}`.
  - **Connections:** Input from `Send Message to OpenAI`, output to `Image Generation Model2`.

- **Image Generation Model2**
  - **Type & Role:** `n8n-nodes-base.httpRequest` — Calls OpenAI Image Edits endpoint (`gpt-image-2`) for branch 3.
  - **Configuration:** URL: `https://api.openai.com/v1/images/edits`, Method: `POST`, Content-Type: `multipart-form-data`.
  - **Connections:** Input from `Download file3`, output to `Convert to File2`.

- **Convert to File2**
  - **Type & Role:** `n8n-nodes-base.convertToFile` — Converts base64 generation response to binary.
  - **Configuration:** Operation: `toBinary`, Source Property: `data[0].b64_json`, Binary Property Name: `image`.
  - **Connections:** Input from `Convert to File2`, output to `Upload file2`.

- **Upload file2**
  - **Type & Role:** `n8n-nodes-base.googleDrive` — Uploads final branch 3 image.
  - **Configuration:** Operation: `upload`, Name: `Content`, Drive ID: `My Drive`, Folder ID: Configured target folder placeholder.
  - **Connections:** Input from `Convert to File2`.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | n8n-nodes-base.stickyNote | Workflow documentation & setup instructions | None | None | ## AI Product Image Generator<br><br>### How it works<br><br>This workflow accepts a product image through an n8n form, stores and retrieves it from Google Drive, then prepares it for AI analysis. It uses OpenAI vision and an AI Agent to interpret the image and create three variation prompts, then each branch calls OpenAI's image edit endpoint to generate a new product image. The generated images are converted back into files and uploaded to Google Drive as separate outputs.<br><br>### Setup steps<br><br>- Configure the form trigger with a required product image upload field.<br>- Connect Google Drive credentials and set the upload/download nodes to the correct source and destination folders.<br>- Configure OpenAI credentials for the OpenAI nodes and ensure the HTTP Request nodes can authenticate against https://api.openai.com/v1/images/edits.<br>- Review the file field names and binary property names across the extract, convert, download, and upload nodes so each branch passes the correct image data.<br><br>### Customization<br><br>Adjust the AI Agent instructions and the three message-model prompts to change the style, background, or marketing angle of generated product images. You can also change the OpenAI image model parameters, output size, number of branches, and Google Drive destination folders. |
| Sticky Note1 | n8n-nodes-base.stickyNote | Captures form image input | None | None | ## Capture form image<br><br>Starts the workflow when a user submits the product image form. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Stages original file in Google Drive | None | None | ## Stage original file<br><br>Uploads the submitted image to Google Drive and downloads it again so downstream file-processing nodes can access a consistent file reference. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Prepares image analysis | None | None | ## Prepare image analysis<br><br>Extracts the downloaded image data, converts it into the expected file format, and sends it to OpenAI for visual analysis. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Plans image variants using AI Agent | None | None | ## Plan image variants<br><br>Uses an AI Agent, OpenAI chat model, and structured output parser to turn the image analysis into structured creative directions for generated product-image variants. |
| Sticky Note5 | n8n-nodes-base.stickyNote | Creates branch prompts | None | None | ## Create branch prompts<br><br>Generates three separate OpenAI model messages from the agent output, one for each product-image generation branch. |
| Sticky Note6 | n8n-nodes-base.stickyNote | Generates first image variation | None | None | ## Generate first image<br><br>Downloads the source image for the first branch, sends it to OpenAI's image edit endpoint, converts the result to a file, and uploads the finished image to Google Drive. |
| Sticky Note7 | n8n-nodes-base.stickyNote | Generates second image variation | None | None | ## Generate second image<br><br>Runs the middle generation branch by downloading the image, calling the OpenAI image edit endpoint, converting the generated output, and saving it to Google Drive. |
| Sticky Note8 | n8n-nodes-base.stickyNote | Generates third image variation | None | None | ## Generate third image<br><br>Runs the lower generation branch by downloading the image, calling the OpenAI image edit endpoint, converting the generated output, and uploading the final image to Google Drive. |
| When Form Submitted | n8n-nodes-base.formTrigger | Receives user input (image, scene brief, gender) | None | Upload file3 | ## Capture form image<br><br>Starts the workflow when a user submits the product image form. |
| Upload file3 | n8n-nodes-base.googleDrive | Uploads submitted form image to Drive root | When Form Submitted | Download file | ## Stage original file<br><br>Uploads the submitted image to Google Drive and downloads it again so downstream file-processing nodes can access a consistent file reference. |
| Download file | n8n-nodes-base.googleDrive | Downloads staged file to get persistent reference ID | Upload file3 | Extract Data from File | ## Stage original file<br><br>Uploads the submitted image to Google Drive and downloads it again so downstream file-processing nodes can access a consistent file reference. |
| Extract Data from File | n8n-nodes-base.extractFromFile | Extracts binary stream from downloaded file | Download file | Convert Data to File | ## Prepare image analysis<br><br>Extracts the downloaded image data, converts it into the expected file format, and sends it to OpenAI for visual analysis. |
| Convert Data to File | n8n-nodes-base.convertToFile | Converts data property to binary file format | Extract Data from File | OpenAI Image Analysis | ## Prepare image analysis<br><br>Extracts the downloaded image data, converts it into the expected file format, and sends it to OpenAI for visual analysis. |
| OpenAI Image Analysis | @n8n/n8n-nodes-langchain.openAi | Analyzes image visual vibe using GPT-4o | Convert Data to File | AI Analysis Agent | ## Prepare image analysis<br><br>Extracts the downloaded image data, converts it into the expected file format, and sends it to OpenAI for visual analysis. |
| OpenAI GPT-4 Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides language model for AI Agent | None | AI Analysis Agent (ai_languageModel) | ## Plan image variants<br><br>Uses an AI Agent, OpenAI chat model, and structured output parser to turn the image analysis into structured creative directions for generated product-image variants. |
| Parse Structured Output | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces JSON output structure for agent | None | AI Analysis Agent (ai_outputParser) | ## Plan image variants<br><br>Uses an AI Agent, OpenAI chat model, and structured output parser to turn the image analysis into structured creative directions for generated product-image variants. |
| AI Analysis Agent | @n8n/n8n-nodes-langchain.agent | Orchestrates scene breakdown and prompt parsing | OpenAI Image Analysis | Message a model1, OpenAI Message Model, Send Message to OpenAI | ## Plan image variants<br><br>Uses an AI Agent, OpenAI chat model, and structured output parser to turn the image analysis into structured creative directions for generated product-image variants. |
| Message a model1 | @n8n/n8n-nodes-langchain.openAi | Refines prompt for branch 1 | AI Analysis Agent | Download file1 | ## Create branch prompts<br><br>Generates three separate OpenAI model messages from the agent output, one for each product-image generation branch. |
| OpenAI Message Model | @n8n/n8n-nodes-langchain.openAi | Refines prompt for branch 2 | AI Analysis Agent | Download file2 | ## Create branch prompts<br><br>Generates three separate OpenAI model messages from the agent output, one for each product-image generation branch. |
| Send Message to OpenAI | @n8n/n8n-nodes-langchain.openAi | Refines prompt for branch 3 | AI Analysis Agent | Download file3 | ## Create branch prompts<br><br>Generates three separate OpenAI model messages from the agent output, one for each product-image generation branch. |
| Download file1 | n8n-nodes-base.googleDrive | Downloads source image for branch 1 | Message a model1 | Post to OpenAI Image Edits | ## Generate first image<br><br>Downloads the source image for the first branch, sends it to OpenAI's image edit endpoint, converts the result to a file, and uploads the finished image to Google Drive. |
| Post to OpenAI Image Edits | n8n-nodes-base.httpRequest | Calls OpenAI images edits API (branch 1) | Download file1 | Convert JSON to File Format | ## Generate first image<br><br>Downloads the source image for the first branch, sends it to OpenAI's image edit endpoint, converts the result to a file, and uploads the finished image to Google Drive. |
| Convert JSON to File Format | n8n-nodes-base.convertToFile | Converts base64 API response to binary (branch 1) | Post to OpenAI Image Edits | Upload File to Google Drive | ## Generate first image<br><br>Downloads the source image for the first branch, sends it to OpenAI's image edit endpoint, converts the result to a file, and uploads the finished image to Google Drive. |
| Upload File to Google Drive | n8n-nodes-base.googleDrive | Uploads final image 1 to Drive | Convert JSON to File Format | None | ## Generate first image<br><br>Downloads the source image for the first branch, sends it to OpenAI's image edit endpoint, converts the result to a file, and uploads the finished image to Google Drive. |
| Download file2 | n8n-nodes-base.googleDrive | Downloads source image for branch 2 | OpenAI Message Model | Image Generation Model1 | ## Generate second image<br><br>Runs the middle generation branch by downloading the image, calling the OpenAI image edit endpoint, converting the generated output, and saving it to Google Drive. |
| Image Generation Model1 | n8n-nodes-base.httpRequest | Calls OpenAI images edits API (branch 2) | Download file2 | Convert to File1 | ## Generate second image<br><br>Runs the middle generation branch by downloading the image, calling the OpenAI image edit endpoint, converting the generated output, and saving it to Google Drive. |
| Convert to File1 | n8n-nodes-base.convertToFile | Converts base64 API response to binary (branch 2) | Image Generation Model1 | Upload file1 | ## Generate second image<br><br>Runs the middle generation branch by downloading the image, calling the OpenAI image edit endpoint, converting the generated output, and saving it to Google Drive. |
| Upload file1 | n8n-nodes-base.googleDrive | Uploads final image 2 to Drive | Convert to File1 | None | ## Generate second image<br><br>Runs the middle generation branch by downloading the image, calling the OpenAI image edit endpoint, converting the generated output, and saving it to Google Drive. |
| Download file3 | n8n-nodes-base.googleDrive | Downloads source image for branch 3 | Send Message to OpenAI | Image Generation Model2 | ## Generate third image<br><br>Runs the lower generation branch by downloading the image, calling the OpenAI image edit endpoint, converting the generated output, and uploading the final image to Google Drive. |
| Image Generation Model2 | n8n-nodes-base.httpRequest | Calls OpenAI images edits API (branch 3) | Download file3 | Convert to File2 | ## Generate third image<br><br>Runs the lower generation branch by downloading the image, calling the OpenAI image edit endpoint, converting the generated output, and uploading the final image to Google Drive. |
| Convert to File2 | n8n-nodes-base.convertToFile | Converts base64 API response to binary (branch 3) | Image Generation Model2 | Convert to File2 (Output: Upload file2) | ## Generate third image<br><br>Runs the lower generation branch by downloading the image, calling the OpenAI image edit endpoint, converting the generated output, and uploading the final image to Google Drive. |
| Upload file2 | n8n-nodes-base.googleDrive | Uploads final image 3 to Drive | Convert to File2 | None | ## Generate third image<br><br>Runs the lower generation branch by downloading the image, calling the OpenAI image edit endpoint, converting the generated output, and uploading the final image to Google Drive. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Form Trigger**
   - Add a **When Form Submitted** (`n8n-nodes-base.formTrigger`) node.
   - Set form title to `3 Image generator`.
   - Add three form fields:
     - Field 1: Label `Scene Brief` (Type: Text)
     - Field 2: Label `Image` (Type: File, Required)
     - Field 3: Label `female/Man` (Type: Text)

2. **Setup Staging in Google Drive**
   - Add an **Upload file3** (`n8n-nodes-base.googleDrive`) node. Connect it to the Form Trigger.
     - Set Operation: `upload`, Drive ID: `My Drive`, Folder: Root (`root`), Input Data Field Name: `Image`.
   - Add a **Download file** (`n8n-nodes-base.googleDrive`) node. Connect it after `Upload file3`.
     - Set Operation: `download`, File ID: `={{ $json.id }}`.

3. **Prepare Visual Analysis**
   - Add an **Extract Data from File** (`n8n-nodes-base.extractFromFile`) node. Operation: `binaryToProperty`. Connect after `Download file`.
   - Add a **Convert Data to File** (`n8n-nodes-base.convertToFile`) node. Operation: `toBinary`, Source Property: `data`. Connect after Extract node.
   - Add an **OpenAI Image Analysis** (`@n8n/n8n-nodes-langchain.openAi`) node. Model: `gpt-4o`, Resource: `image`, Input Type: `base64`, Operation: `analyze`. Connect after Convert node. Requires OpenAI API credentials.

4. **Configure AI Planning Agent**
   - Add an **AI Analysis Agent** (`@n8n/n8n-nodes-langchain.agent`) node. Prompt Type: `define`. Connect input from OpenAI Image Analysis.
   - Add an **OpenAI GPT-4 Chat Model** (`@n8n/n8n-nodes-langchain.lmChatOpenAi`) node. Model: `gpt-4.1-mini`. Connect to agent via `ai_languageModel`.
   - Add a **Parse Structured Output** (`@n8n/n8n-nodes-langchain.outputParserStructured`) node. Provide the JSON schema defining an array of three scene strings. Connect to agent via `ai_outputParser`.

5. **Build Branch 1 (Primary Output)**
   - Add **Message a model1** (`@n8n/n8n-nodes-langchain.openAi`) node (Model: `gpt-4.1`). Connect to agent output.
   - Add **Download file1** (`n8n-nodes-base.googleDrive`) node (Operation: `download`, File ID: `={{ $('Download file').first().json.id }}`). Connect after message model.
   - Add **Post to OpenAI Image Edits** (`n8n-nodes-base.httpRequest`) node. URL: `https://api.openai.com/v1/images/edits`, Method: `POST`, Content-Type: `multipart-form-data`. Add body parameters: `prompt` (with Quiet Luxury styling), `image[]` (formBinaryData from `data`), `model` (`gpt-image-2`), `size` (`1024x1536`), `quality` (`medium`). Requires OpenAI API credentials. Connect after `Download file1`.
   - Add **Convert JSON to File Format** (`n8n-nodes-base.convertToFile`) node (Operation: `toBinary`, Source Property: `data[0].b64_json`, Binary Property Name: `image`). Connect after HTTP request.
   - Add **Upload File to Google Drive** (`n8n-nodes-base.googleDrive`) node (Operation: `upload`, Input Field: `image`, Target Folder: Select destination folder ID). Connect after conversion node. Requires Google Drive OAuth2 credentials.

6. **Build Branch 2 (Secondary Output)**
   - Add **OpenAI Message Model** (`@n8n/n8n-nodes-langchain.openAi`) node (Model: `gpt-4o`). Connect to agent output.
   - Add **Download file2** (`n8n-nodes-base.googleDrive`) node (Operation: `download`, File ID: `={{ $('Download file').first().json.id }}`).
   - Add **Image Generation Model1** (`n8n-nodes-base.httpRequest`) node configured identically to Branch 1's HTTP request, utilizing branch 2 prompt text.
   - Add **Convert to File1** (`n8n-nodes-base.convertToFile`) node (Operation: `toBinary`, Source Property: `data[0].b64_json`, Binary Property Name: `image`).
   - Add **Upload file1** (`n8n-nodes-base.googleDrive`) node (Operation: `upload`, Input Field: `image`, Target Folder: Select destination folder ID).

7. **Build Branch 3 (Tertiary Output)**
   - Add **Send Message to OpenAI** (`@n8n/n8n-nodes-langchain.openAi`) node (Model: `gpt-4o`). Connect to agent output.
   - Add **Download file3** (`n8n-nodes-base.googleDrive`) node (Operation: `download`, File ID: `={{ $('Download file').first().json.id }}`).
   - Add **Image Generation Model2** (`n8n-nodes-base.httpRequest`) node configured identically to Branch 1 & 2's HTTP requests.
   - Add **Convert to File2** (`n8n-nodes-base.convertToFile`) node (Operation: `toBinary`, Source Property: `data[0].b64_json`, Binary Property Name: `image`).
   - Add **Upload file2** (`n8n-nodes-base.googleDrive`) node (Operation: `upload`, Input Field: `image`, Target Folder: Select destination folder ID).

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Google Drive Destination Folders Configuration | The workflow contains template placeholders (`Select your Google Drive folder here`) in all final upload nodes. These must be replaced with valid Google Drive folder IDs before execution. |
| OpenAI API Authentication | Requires a valid OpenAI API credential with permissions to access GPT-4o, GPT-4.1-mini, and the image editing endpoint (`gpt-image-2`). |
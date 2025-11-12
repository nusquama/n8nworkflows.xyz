Generate Logos and Images with Consistent Visual Styles using Imagen 3.0

https://n8nworkflows.xyz/workflows/generate-logos-and-images-with-consistent-visual-styles-using-imagen-3-0-3954


# Generate Logos and Images with Consistent Visual Styles using Imagen 3.0

### 1. Workflow Overview

This workflow automates the generation of logos or images that mimic the visual style of a provided source image, using Google’s Gemini 2.0 and Imagen 3.0 AI models. It targets use cases in design and marketing automation, enabling users to quickly create stylistic variants or remixed assets without manual design effort.

The workflow is logically divided into these blocks:

- **1.1 Input Reception and Validation:** Captures user inputs via a form, validating the source image URL.
- **1.2 Visual Style Extraction:** Downloads the source image and uses Gemini 2.0 to analyze and describe its visual style.
- **1.3 Image Generation:** Combines the style description with the user’s prompt and uses Imagen 3.0 to generate new images.
- **1.4 Image Processing and Upload:** Converts generated images, uploads them to Cloudinary CDN for reliable hosting.
- **1.5 Result Presentation:** Generates an HTML page displaying the generated images along with style details.
- **1.6 Optional Email Delivery:** Sends the HTML page to the user’s email if provided.
- **1.7 Completion and Download:** Provides a downloadable HTML file upon workflow completion.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Validation

**Overview:**  
This block collects user input via a form trigger and validates that the source image URL is valid, looping back if not.

**Nodes Involved:**  
- On form submission  
- Form Validation  
- Retry Form  
- Variables

**Node Details:**  

- **On form submission**  
  - Type: Form Trigger  
  - Role: Entry point capturing user inputs: source image URL, target prompt, number of images, optional email.  
  - Config: Form path `/style-copy-with-imagen3`, button label "Generate!", fields with validation requirements, response mode set to last node.  
  - Inputs: External user form submission  
  - Outputs: To Form Validation  
  - Edge cases: Submission without required fields; bots ignored by config.

- **Form Validation**  
  - Type: If (conditional) node  
  - Role: Checks if "SourceImage" field is a valid URL.  
  - Config: Expression `{{$json.SourceImage.isUrl()}}` tested for true.  
  - Inputs: On form submission  
  - Outputs: True → Variables node; False → Retry Form node  
  - Edge cases: Invalid or malformed URLs, empty inputs triggering retry.

- **Retry Form**  
  - Type: Form node  
  - Role: Redisplays form for user to correct invalid source image URL.  
  - Config: Same form fields as original, with message prompting for valid URL.  
  - Inputs: Form Validation false branch  
  - Outputs: Back to Form Validation (loop)  
  - Edge cases: User repeatedly submits invalid data or abandons.

- **Variables**  
  - Type: Set node  
  - Role: Extracts and sanitizes key variables from form input for downstream use.  
  - Config:  
    - `sourceStyleUrl` from `SourceImage`  
    - `targetPrompt` from `TargetPrompt`  
    - `numberSamples` coerced to integer between 1 and 4 (default 1 if invalid)  
    - `email` from optional field "Your Email (Optional)"  
  - Inputs: Form Validation true branch  
  - Outputs: Download Image node  
  - Edge cases: Missing or out-of-range "Number of Images" handled by function.

---

#### 1.2 Visual Style Extraction

**Overview:**  
Downloads the source image and uses Google Gemini 2.0 multimodal model to generate a detailed description of the image’s visual style.

**Nodes Involved:**  
- Download Image  
- Image to Base64  
- Gemini 2.0

**Node Details:**  

- **Download Image**  
  - Type: HTTP Request  
  - Role: Fetches the source image binary data from the user-provided URL.  
  - Config: URL set dynamically from `Variables.sourceStyleUrl`.  
  - Inputs: Variables node  
  - Outputs: Image to Base64  
  - Edge cases: URL inaccessible, timeout, invalid image data, HTTP errors.

- **Image to Base64**  
  - Type: ExtractFromFile  
  - Role: Converts downloaded binary image data into base64-encoded string property for Gemini consumption.  
  - Config: Operation "binaryToProperty", input property `data`.  
  - Inputs: Download Image  
  - Outputs: Gemini 2.0  
  - Edge cases: Corrupt binary data causing conversion failure.

- **Gemini 2.0**  
  - Type: HTTP Request  
  - Role: Calls Gemini 2.0 model with the base64 image data to generate a textual visual style description.  
  - Config:  
    - POST to `gemini-2.0-flash:generateContent` endpoint  
    - Request body includes inline image data with mime type and base64 content, plus prompt text asking for style description excluding character names/IP, including artist names if known.  
    - Uses Google Palm API credentials.  
  - Inputs: Image to Base64  
  - Outputs: Imagen 3.0  
  - Edge cases: API rate limits, auth errors, image too large or unsupported format, malformed JSON response.

---

#### 1.3 Image Generation

**Overview:**  
Generates new images based on a combined prompt of the Gemini style description and the user's target prompt using Google Imagen 3.0 model.

**Nodes Involved:**  
- Imagen 3.0  
- Split Out

**Node Details:**  

- **Imagen 3.0**  
  - Type: HTTP Request  
  - Role: Sends a prompt to Imagen 3.0 for image generation.  
  - Config:  
    - POST to Imagen 3.0 generate endpoint  
    - Prompt is a combination of Gemini's style text plus user target prompt:  
      `"{Gemini style description} Generate the following image: {targetPrompt}"`  
    - `sampleCount` set from `Variables.numberSamples` (1-4)  
    - Auth via Google Palm API credentials  
  - Inputs: Gemini 2.0  
  - Outputs: Split Out  
  - Edge cases: API limits, invalid prompt formatting, timeout, auth errors.

- **Split Out**  
  - Type: Split Out  
  - Role: Separates array of generated image predictions into individual outputs for further processing.  
  - Config: Splits on `predictions` property.  
  - Inputs: Imagen 3.0  
  - Outputs: Convert to File  
  - Edge cases: Empty or malformed predictions array.

---

#### 1.4 Image Processing and Upload

**Overview:**  
Converts each generated image from base64 to binary file, uploads it to Cloudinary CDN for hosting.

**Nodes Involved:**  
- Convert to File  
- Upload to Cloudinary

**Node Details:**  

- **Convert to File**  
  - Type: ConvertToFile  
  - Role: Converts base64-encoded image bytes from Imagen into binary file format suitable for upload.  
  - Config:  
    - Filename dynamically set to `{executionId}_{itemIndex}.{mimeType extension}`  
    - mimeType taken from JSON property  
  - Inputs: Split Out  
  - Outputs: Upload to Cloudinary  
  - Edge cases: Missing mimeType, corrupt base64 data.

- **Upload to Cloudinary**  
  - Type: HTTP Request  
  - Role: Uploads converted image binary to Cloudinary using preset for anonymous upload.  
  - Config:  
    - POST to Cloudinary image upload endpoint  
    - Multipart form data with binary file field named `file`  
    - Query param `upload_preset=n8n-workflows-preset`  
    - Auth via generic HTTP Query Auth credentials (Cloudinary API)  
  - Inputs: Convert to File  
  - Outputs: Generate HTML  
  - Edge cases: Upload failure, credential errors, network issues.

---

#### 1.5 Result Presentation

**Overview:**  
Generates an HTML page showing the generated images in a gallery format alongside the style description and source image.

**Nodes Involved:**  
- Generate HTML  
- Convert to File1

**Node Details:**  

- **Generate HTML**  
  - Type: HTML  
  - Role: Produces a stylized HTML page with:  
    - User's target prompt as title  
    - Gallery showing generated images (linked and thumbnails) in rows of two  
    - Fine print with generation date and style prompt details including source thumbnail  
  - Config: HTML template using expressions referencing Cloudinary URLs, Gemini style text, and current date.  
  - Inputs: Upload to Cloudinary  
  - Outputs: Convert to File1, Has Email?  
  - Edge cases: Missing image URLs, empty gallery, HTML rendering issues.

- **Convert to File1**  
  - Type: ConvertToFile  
  - Role: Converts generated HTML text into downloadable UTF-8 encoded `.html` file named from sanitized target prompt.  
  - Config: Filename derived by transforming target prompt to snake case and URL encoding.  
  - Inputs: Generate HTML  
  - Outputs: Form Ending  
  - Edge cases: Filename conflicts, encoding issues.

---

#### 1.6 Optional Email Delivery

**Overview:**  
If user provided an email, sends generated HTML page as email content via Gmail.

**Nodes Involved:**  
- Has Email?  
- Send Results to Email

**Node Details:**  

- **Has Email?**  
  - Type: If node  
  - Role: Checks if `email` variable is non-empty.  
  - Config: Expression checks `Variables.email` for non-empty string.  
  - Inputs: Generate HTML  
  - Outputs: True → Send Results to Email; False → no further output  
  - Edge cases: Invalid email format not explicitly checked.

- **Send Results to Email**  
  - Type: Gmail node  
  - Role: Sends email with subject including execution ID and body containing generated HTML page.  
  - Config:  
    - SendTo from `Variables.email`  
    - Message set to HTML content from previous node  
    - Subject includes execution ID and success message  
    - Uses Gmail OAuth2 credentials  
  - Inputs: Has Email? true branch  
  - Outputs: None (end)  
  - Edge cases: Gmail API limits, auth expiration, invalid recipient email.

---

#### 1.7 Completion and Download

**Overview:**  
Provides a final form node signaling successful generation and triggers download of the HTML file.

**Nodes Involved:**  
- Form Ending

**Node Details:**  

- **Form Ending**  
  - Type: Form node  
  - Role: Completion message and automatic download trigger for HTML file.  
  - Config:  
    - Title: "Generation Complete"  
    - Completion message: "Download has started. Open the HTML file to view results."  
    - Responds with binary file to initiate download  
    - Executes once per workflow run  
  - Inputs: Convert to File1  
  - Outputs: None (workflow end)  
  - Edge cases: Browser compatibility for download, user aborts download.

---

### 3. Summary Table

| Node Name           | Node Type           | Functional Role                           | Input Node(s)           | Output Node(s)             | Sticky Note                                                                                                                                                |
|---------------------|---------------------|-----------------------------------------|------------------------|----------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------|
| On form submission   | Form Trigger        | Capture user inputs via form             | -                      | Form Validation            | ## 1. Ask for Source Style and Target Image [Learn more about the Form Trigger](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.formtrigger/) |
| Form Validation     | If                  | Validate source image URL                 | On form submission     | Variables, Retry Form       | (See above)                                                                                                                                                 |
| Retry Form          | Form                | Request corrected inputs if invalid URL  | Form Validation        | Form Validation            | (See above)                                                                                                                                                 |
| Variables           | Set                 | Extract and sanitize key variables       | Form Validation        | Download Image             | (See above)                                                                                                                                                 |
| Download Image      | HTTP Request        | Download source image binary              | Variables              | Image to Base64            | ## 2. Visual Style Description using Gemini 2.0 [Read about Gemini Image Understanding](https://ai.google.com/gemini-api/docs/image-understanding)          |
| Image to Base64     | ExtractFromFile     | Convert binary image to base64            | Download Image         | Gemini 2.0                 | (See above)                                                                                                                                                 |
| Gemini 2.0          | HTTP Request        | Generate visual style description         | Image to Base64        | Imagen 3.0                 | (See above)                                                                                                                                                 |
| Imagen 3.0          | HTTP Request        | Generate images from combined prompt      | Gemini 2.0             | Split Out                  | ## 3. Image Generation using Imagen 3.0 [Read about Imagen Image Generation](https://ai.google.com/gemini-api/docs/image-generation#imagen)                |
| Split Out           | Split Out           | Split image array into individual images  | Imagen 3.0             | Convert to File            | (See above)                                                                                                                                                 |
| Convert to File     | ConvertToFile       | Convert base64 images to binary files     | Split Out              | Upload to Cloudinary       | (See above)                                                                                                                                                 |
| Upload to Cloudinary| HTTP Request        | Upload images to Cloudinary CDN            | Convert to File        | Generate HTML              | ## 4. Render Results to HTML Page [Learn about the HTML node](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.html)                      |
| Generate HTML       | HTML                | Create HTML gallery page with images      | Upload to Cloudinary   | Convert to File1, Has Email? | (See above)                                                                                                                                                 |
| Convert to File1    | ConvertToFile       | Convert HTML text to downloadable file    | Generate HTML          | Form Ending                | (See above)                                                                                                                                                 |
| Has Email?          | If                  | Check if user provided email              | Generate HTML          | Send Results to Email      | (See above)                                                                                                                                                 |
| Send Results to Email| Gmail               | Email generated HTML page to user         | Has Email?             | -                          | (See above)                                                                                                                                                 |
| Form Ending         | Form                | Completion message and download trigger   | Convert to File1       | -                          | (See above)                                                                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger node "On form submission"**  
   - Set webhook path: `/style-copy-with-imagen3`  
   - Configure form fields:  
     - SourceImage (string, required)  
     - TargetPrompt (string, required)  
     - Number of Images (number, optional, max 4)  
     - Your Email (optional, email)  
   - Set button label: "Generate!"  
   - Enable "Ignore bots"  
   - Response mode: last node

2. **Add If node "Form Validation"**  
   - Condition: `{{$json.SourceImage.isUrl()}}` is true  
   - Connect "On form submission" → "Form Validation"

3. **Add Form node "Retry Form"**  
   - Same fields as original form  
   - Button label: "Generate!"  
   - Description: "Please enter a URL for the source image."  
   - Connect "Form Validation" false output → "Retry Form"  
   - Connect "Retry Form" output → "Form Validation" input (loop)

4. **Add Set node "Variables"**  
   - Assignments:  
     - `sourceStyleUrl` = `{{$json.SourceImage}}`  
     - `targetPrompt` = `{{$json.TargetPrompt}}`  
     - `numberSamples` = function: clamp `Number of Images` between 1 and 4, default 1  
     - `email` = `{{$json['Your Email (Optional)']}}`  
   - Connect "Form Validation" true output → "Variables"

5. **Add HTTP Request node "Download Image"**  
   - Method: GET  
   - URL: `={{ $json.sourceStyleUrl }}`  
   - Connect "Variables" → "Download Image"

6. **Add ExtractFromFile node "Image to Base64"**  
   - Operation: binaryToProperty  
   - Source property: `data` (the binary file from "Download Image")  
   - Connect "Download Image" → "Image to Base64"

7. **Add HTTP Request node "Gemini 2.0"**  
   - URL: Gemini 2.0 generateContent endpoint  
   - Method: POST  
   - Auth: Google Palm API credentials  
   - Body (JSON):  
     - Include inline base64 image data property from "Image to Base64" and prompt asking for visual style description excluding IP/character names, including artist names if known  
   - Connect "Image to Base64" → "Gemini 2.0"

8. **Add HTTP Request node "Imagen 3.0"**  
   - URL: Imagen 3.0 generate endpoint  
   - Method: POST  
   - Auth: Google Palm API credentials  
   - Body (JSON):  
     - Prompt: Combine Gemini 2.0 style description + "Generate the following image: {targetPrompt}"  
     - sampleCount from `Variables.numberSamples`  
   - Connect "Gemini 2.0" → "Imagen 3.0"

9. **Add Split Out node "Split Out"**  
   - Field to split: `predictions` (array of generated images)  
   - Connect "Imagen 3.0" → "Split Out"

10. **Add ConvertToFile node "Convert to File"**  
    - Operation: toBinary  
    - Filename: `{executionId}_{itemIndex}.{mimeType extension}` (dynamically from JSON)  
    - MimeType from JSON  
    - Connect "Split Out" → "Convert to File"

11. **Add HTTP Request node "Upload to Cloudinary"**  
    - Method: POST  
    - URL: Cloudinary upload endpoint  
    - Auth: Cloudinary API credentials (HTTP Query Auth)  
    - Body: multipart-form-data with binary file field named `file`  
    - Query parameter: upload_preset = `n8n-workflows-preset`  
    - Connect "Convert to File" → "Upload to Cloudinary"

12. **Add HTML node "Generate HTML"**  
    - Compose HTML page with:  
      - Title: targetPrompt (sentence cased)  
      - Gallery: images from Cloudinary URLs, grouped in rows of 2  
      - Fine print: generation date, style prompt description, thumbnail of source image  
      - Include CSS styling for gallery layout  
    - Connect "Upload to Cloudinary" → "Generate HTML"

13. **Add ConvertToFile node "Convert to File1"**  
    - Operation: toText  
    - Encoding: UTF-8  
    - Filename: sanitized targetPrompt converted to snake_case and URL encoded + `.html`  
    - Connect "Generate HTML" → "Convert to File1"

14. **Add If node "Has Email?"**  
    - Condition: `Variables.email` is not empty string  
    - Connect "Generate HTML" → "Has Email?"

15. **Add Gmail node "Send Results to Email"**  
    - Recipient: `Variables.email`  
    - Subject: `#{{$execution.id}} - Image Generated Successfully!`  
    - Message: HTML content from "Generate HTML"  
    - Auth: Gmail OAuth2 credentials  
    - Connect "Has Email?" true output → "Send Results to Email"

16. **Add Form node "Form Ending"**  
    - Operation: Completion  
    - Title: "Generation Complete"  
    - Completion message: "Download has started. Open the HTML file to view results."  
    - Respond with: binary file from "Convert to File1"  
    - Connect "Convert to File1" → "Form Ending"

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                             | Context or Link                                                                                                  |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| The form trigger node allows easy public sharing and user input collection. Consider switching to webhook trigger for full automation.                                                  | [n8n Form Trigger Docs](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.formtrigger/)        |
| Gemini 2.0 provides advanced multimodal image understanding to extract detailed style descriptions crucial for style transfer tasks.                                                    | [Gemini Image Understanding](https://ai.google.dev/gemini-api/docs/image-understanding)                         |
| Imagen 3.0 is Google’s state-of-the-art text-to-image generation model, capable of incorporating style descriptions with user prompts.                                                  | [Imagen Prompt Guide](https://ai.google.dev/gemini-api/docs/image-generation#imagen-prompt-guide)                |
| Cloudinary is used as a CDN for reliable, fast hosting of generated images, allowing easy embedding in the HTML results page.                                                             | [Cloudinary Upload API](https://cloudinary.com/documentation/image_upload_api_reference)                         |
| Gmail node requires OAuth2 credentials configured correctly for sending emails. Ensure the Gmail account has appropriate API access enabled.                                            | [n8n Gmail Node Docs](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.gmail/)                   |
| The HTML results page includes interactive image gallery and style attribution, improving user experience.                                                                               | [n8n HTML Node Docs](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.html/)                    |
| Join the n8n Community on Discord or Forum for support and ideas.                                                                                                                        | [Discord](https://discord.com/invite/XPKeKXeB7d), [Forum](https://community.n8n.io/)                             |

---

This document provides a thorough understanding and reproduction guide for the "Generate Logos and Images with Consistent Visual Styles using Imagen 3.0" workflow, enabling advanced users and AI agents to manage and extend it confidently.
Design and Post LinkedIn Content with AI Captions and Custom Templates

https://n8nworkflows.xyz/workflows/design-and-post-linkedin-content-with-ai-captions-and-custom-templates-11620


# Design and Post LinkedIn Content with AI Captions and Custom Templates

### 1. Workflow Overview

This n8n workflow automates the creation and posting of designed LinkedIn content using AI-generated captions and custom HTML-based image templates. It targets marketing professionals and content creators who want to streamline the process of generating visually appealing posts with motivational or inspirational quotes and directly publish them on LinkedIn.

The workflow is logically divided into three main blocks:

- **1.1 Input Reception and Data Processing:** Receives form submissions containing a portrait image and a quote, parses and converts the image, and generates an HTML design with one of six randomized templates embedding the quote and image.
  
- **1.2 Image Generation and Caption Creation:** Converts the HTML design to an image via an external API, downloads and resizes the image, then generates a compelling LinkedIn caption using OpenAI's ChatGPT API based on the quote.

- **1.3 LinkedIn Posting and Response:** Provides an endpoint to receive a final post request, creates the LinkedIn post via API, and sends back a success confirmation.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Data Processing

- **Overview:**  
  This block handles the initial input from a webform submission, parses the binary image and quote, converts the image to base64, selects a random HTML design template embedding the image and quote, and prepares it for image conversion.

- **Nodes Involved:**  
  - Webhook - Receive Form  
  - Parse Form Data  
  - Create HTML Design

- **Node Details:**

  - **Webhook - Receive Form**  
    - *Type & Role:* Webhook node, entry point to receive POST requests with form data.  
    - *Configuration:*  
      - Path: Unique webhook path for form submission.  
      - Method: POST.  
      - Raw body enabled to capture binary image data.  
      - Response mode set to "lastNode" to return final processed result.  
    - *Input/Output:* Receives multipart/form-data with an image and quote; outputs raw payload for parsing.  
    - *Edge Cases:* Missing image or quote data can cause errors downstream.  
    - *Version:* 1.1

  - **Parse Form Data**  
    - *Type & Role:* Code node, extracts and transforms incoming webhook data.  
    - *Configuration:*  
      - Extracts quote from JSON body parameters or defaults to "No quote provided."  
      - Extracts image binary data, converts to base64, and detects MIME type.  
    - *Key Expressions:* Uses `$input.item.binary` for image and `$input.item.json.body` for quote.  
    - *Input/Output:* Input from webhook; outputs JSON with quote, base64 image, MIME type, and timestamp.  
    - *Failure Modes:* Throws error if no image uploaded, which stops workflow.  
    - *Version:* 2

  - **Create HTML Design**  
    - *Type & Role:* Code node, generates a full HTML document with embedded base64 image and quote using one of six randomized templates.  
    - *Configuration:*  
      - Randomly selects a design index (0–5) based on timestamp and execution ID for better randomness.  
      - Each design includes CSS styling for backgrounds, image circles, and quote display with unique color schemes and layout.  
      - Returns the HTML string, the quote, design index and name, and timestamp.  
    - *Key Expressions:* Uses `$json.quote`, `$json.imageBase64`, and `$json.mimeType` as inputs.  
    - *Input/Output:* Input from Parse Form Data; outputs JSON with generated HTML and metadata.  
    - *Edge Cases:* Randomization ensures variety; if inputs are malformed, may output invalid HTML.  
    - *Version:* 2

---

#### 2.2 Image Generation and Caption Creation

- **Overview:**  
  Converts the generated HTML to a high-resolution image using the external HTMLCSSToImage API, downloads and resizes the image, then generates a LinkedIn caption using OpenAI’s GPT model based on the quote.

- **Nodes Involved:**  
  - HTML to Image (HCTI)  
  - Download Image  
  - Edit Image  
  - Prepare Image Data  
  - ChatGPT - Generate Caption  
  - Prepare Final Response

- **Node Details:**

  - **HTML to Image (HCTI)**  
    - *Type & Role:* HTTP Request node, sends HTML to an external API to convert to image URL.  
    - *Configuration:*  
      - POST to `https://hcti.io/v1/image` with basic HTTP authentication.  
      - Sends HTML content, viewport width/height parameters (1080x1080).  
      - Requires HTMLCSSToImage account for API credentials.  
    - *Input/Output:* Input HTML from Create HTML Design; outputs JSON with image URL.  
    - *Failure Modes:* Auth failure if credentials invalid, API quota limits, or malformed HTML causing conversion failure.  
    - *Version:* 4.1

  - **Download Image**  
    - *Type & Role:* HTTP Request node, downloads the generated image from URL into binary data.  
    - *Configuration:*  
      - GET request to the URL provided by previous node.  
      - Response format set to file/binary.  
    - *Input/Output:* Input URL from previous node; outputs binary image data.  
    - *Edge Cases:* Network timeouts or 404 if URL expired.  
    - *Version:* 4.1

  - **Edit Image**  
    - *Type & Role:* Edit Image node, resizes the downloaded image to optimize size/quality.  
    - *Configuration:*  
      - Operation: Resize (default parameters).  
    - *Input/Output:* Input binary image from Download Image; outputs resized binary image.  
    - *Edge Cases:* Corrupted image data could cause failure.  
    - *Version:* 1

  - **Prepare Image Data**  
    - *Type & Role:* Code node, converts resized binary image to base64 and forwards quote and image URL.  
    - *Configuration:*  
      - Converts binary to base64 string.  
      - Passes quote from parsed form data for caption generation.  
    - *Input/Output:* Input from Edit Image and Parse Form Data; outputs JSON with base64 image, quote, and image URL.  
    - *Version:* 2

  - **ChatGPT - Generate Caption**  
    - *Type & Role:* HTTP Request node, calls OpenAI Chat Completions API to create a LinkedIn caption.  
    - *Configuration:*  
      - POST to `https://api.openai.com/v1/chat/completions`.  
      - Model: `gpt-4o-mini`.  
      - Message: System prompt defines expert LinkedIn caption writer tone and user prompt passes the quote with strict format and style constraints (no em dash, length 100-150 words, with hashtags).  
      - Temperature 0.8, max tokens 400.  
      - Requires OpenAI API key in Authorization header.  
    - *Input/Output:* Input quote from Prepare Image Data; outputs JSON with AI-generated caption.  
    - *Failure Modes:* API key issues, rate limits, or malformed prompt could cause errors or incomplete captions.  
    - *Version:* 4.1

  - **Prepare Final Response**  
    - *Type & Role:* Code node, consolidates caption and design data to prepare JSON response for the webform.  
    - *Configuration:*  
      - Extracts caption safely with try/catch to handle failures gracefully.  
      - Returns success status, quote, base64 image, caption text, and ISO timestamp.  
    - *Input/Output:* Input from ChatGPT and Prepare Image Data; outputs JSON for client consumption.  
    - *Version:* 2

---

#### 2.3 LinkedIn Posting and Response

- **Overview:**  
  Handles an independent webhook that receives finalized post content and creates a LinkedIn post using the LinkedIn API, then responds with a success confirmation.

- **Nodes Involved:**  
  - Webhook-LinkedIn-post  
  - Create a post  
  - Respond to Webhook

- **Node Details:**

  - **Webhook-LinkedIn-post**  
    - *Type & Role:* Webhook node, endpoint to receive LinkedIn post requests from frontend or webform.  
    - *Configuration:*  
      - POST method.  
      - Path uniquely defined.  
      - Response mode: responseNode (waits for downstream response).  
    - *Input/Output:* Receives JSON with post data; passes to Create a post node.  
    - *Version:* 2.1

  - **Create a post**  
    - *Type & Role:* LinkedIn node, posts content to LinkedIn using OAuth2 credentials.  
    - *Configuration:*  
      - Uses LinkedIn OAuth2 credentials configured in n8n.  
      - Post content passed from webhook input.  
      - Supports additional fields if needed (empty here).  
    - *Input/Output:* Input from Webhook-LinkedIn-post; output triggers response node.  
    - *Failure Modes:* Auth token expiration, API rate limits, or invalid post data.  
    - *Version:* 1

  - **Respond to Webhook**  
    - *Type & Role:* Respond to Webhook node, sends HTTP 200 with JSON success message.  
    - *Configuration:*  
      - Response code 200.  
      - Response headers enable CORS for all origins and allow POST method with Content-Type.  
      - Response body JSON with success and message.  
    - *Input/Output:* Input from Create a post node; returns response to caller.  
    - *Version:* 1.4

---

### 3. Summary Table

| Node Name               | Node Type              | Functional Role                                    | Input Node(s)           | Output Node(s)            | Sticky Note                                                                                                         |
|-------------------------|------------------------|---------------------------------------------------|-------------------------|---------------------------|---------------------------------------------------------------------------------------------------------------------|
| Webhook - Receive Form   | Webhook                | Receives form submission with quote and image    |                         | Parse Form Data            | Automated LinkedIn Post Designer with N8N Integration; setup instructions and webform link included                 |
| Parse Form Data          | Code                   | Parses input, extracts quote and base64 image     | Webhook - Receive Form  | Create HTML Design         |                                                                                                                     |
| Create HTML Design       | Code                   | Generates HTML design with randomized template    | Parse Form Data          | HTML to Image (HCTI)       |                                                                                                                     |
| HTML to Image (HCTI)     | HTTP Request           | Converts HTML to image via external API           | Create HTML Design       | Download Image             | Sign up at htmlcsstoimage.com - Free 50 images/month                                                                |
| Download Image           | HTTP Request           | Downloads image binary from URL                    | HTML to Image (HCTI)     | Edit Image                 |                                                                                                                     |
| Edit Image              | Edit Image             | Resizes image for optimization                     | Download Image           | Prepare Image Data         |                                                                                                                     |
| Prepare Image Data       | Code                   | Converts image binary to base64, passes quote     | Edit Image               | ChatGPT - Generate Caption |                                                                                                                     |
| ChatGPT - Generate Caption | HTTP Request         | Generates LinkedIn caption via OpenAI API         | Prepare Image Data       | Prepare Final Response     |                                                                                                                     |
| Prepare Final Response   | Code                   | Consolidates caption and image data for response  | ChatGPT - Generate Caption | Respond to Webhook1       |                                                                                                                     |
| Respond to Webhook1      | Respond to Webhook     | Returns JSON response to initial webhook          | Prepare Final Response   |                           |                                                                                                                     |
| Webhook-LinkedIn-post    | Webhook                | Receives post submission for LinkedIn             |                         | Create a post              |                                                                                                                     |
| Create a post            | LinkedIn               | Posts content on LinkedIn                          | Webhook-LinkedIn-post    | Respond to Webhook         |                                                                                                                     |
| Respond to Webhook       | Respond to Webhook     | Sends success response after posting               | Create a post            |                           |                                                                                                                     |
| Sticky Note              | Sticky Note            | Section 3: Confirm and post directly to LinkedIn  |                         |                           |                                                                                                                     |
| Sticky Note1             | Sticky Note            | Overview and setup instructions with webform link |                         |                           | Includes [Link to the webform](https://drive.google.com/file/d/1c2fuxuJXt-eV3qM4aW48H0CVebA1js1j/view?usp=sharing) |
| Sticky Note2             | Sticky Note            | Section 1: Receive data from webform, process       |                         |                           |                                                                                                                     |
| Sticky Note4             | Sticky Note            | Section 2: Generate caption and send response      |                         |                           |                                                                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook - Receive Form**  
   - Type: Webhook  
   - Method: POST  
   - Path: Unique string (e.g., "67adcc76-29c9-412a-b3ce-f34d2944a09f")  
   - Options: Enable Raw Body to capture binary data  
   - Response Mode: Last Node  
   - No credentials required

2. **Add Code Node - Parse Form Data**  
   - Extract quote from `json.body.quote`, default to "No quote provided"  
   - Extract image binary from `binary.image`  
   - Convert binary image data to base64 string  
   - Detect MIME type or default to `image/jpeg`  
   - Output JSON: `{quote, imageBase64, mimeType, timestamp}`  
   - Connect input from Webhook - Receive Form

3. **Add Code Node - Create HTML Design**  
   - Use JavaScript to randomly select one of six HTML templates embedding base64 image and quote  
   - Templates include CSS styling for various themes (Purple Gradient, Dark Elegant, Pink Gradient, Ocean Blue, Teal Modern, Sunset Orange)  
   - Output JSON: `{html, quote, designUsed, designName, timestamp}`  
   - Connect input from Parse Form Data

4. **Add HTTP Request Node - HTML to Image (HCTI)**  
   - Method: POST  
   - URL: `https://hcti.io/v1/image`  
   - Body: `html` from previous node, `viewport_width` 1080, `viewport_height` 1080  
   - Authentication: Basic Auth with HTMLCSSToImage credentials  
   - Connect input from Create HTML Design

5. **Add HTTP Request Node - Download Image**  
   - Method: GET  
   - URL: `url` field from previous node's JSON response  
   - Response Format: File (binary)  
   - Connect input from HTML to Image (HCTI)

6. **Add Edit Image Node - Edit Image**  
   - Operation: Resize (default parameters)  
   - Connect input from Download Image

7. **Add Code Node - Prepare Image Data**  
   - Convert binary image from Edit Image node to base64 string  
   - Pass along quote from Parse Form Data node  
   - Output JSON with `quote`, `designImageBase64`, and `imageUrl`  
   - Connect input from Edit Image and reference quote from Parse Form Data

8. **Add HTTP Request Node - ChatGPT - Generate Caption**  
   - Method: POST  
   - URL: `https://api.openai.com/v1/chat/completions`  
   - Headers: Authorization `Bearer YOUR_TOKEN_HERE`, Content-Type `application/json`  
   - Body parameters:  
     - model: `gpt-4o-mini`  
     - messages: system prompt defining expert LinkedIn caption writer + user prompt including quote and formatting instructions  
     - temperature: 0.8  
     - max_tokens: 400  
   - Connect input from Prepare Image Data

9. **Add Code Node - Prepare Final Response**  
   - Extract caption safely from ChatGPT response  
   - Combine quote, base64 image, caption, and timestamp into JSON response  
   - Connect input from ChatGPT - Generate Caption and Prepare Image Data

10. **Add Respond to Webhook Node - Respond to Webhook1**  
    - Respond with JSON prepared in previous node  
    - Enable CORS headers (`Access-Control-Allow-Origin: *`)  
    - Connect input from Prepare Final Response

11. **Create Second Webhook - Webhook-LinkedIn-post**  
    - Method: POST  
    - Unique path (e.g., "087bc7d9-39b4-48c2-add3-2e73b082cdf4")  
    - Response mode: responseNode  
    - This serves as an endpoint to receive posts ready for LinkedIn

12. **Add LinkedIn Node - Create a post**  
    - Use LinkedIn OAuth2 credentials configured in n8n  
    - Pass post content (caption, image, etc.) from Webhook-LinkedIn-post input  
    - Connect input from Webhook-LinkedIn-post

13. **Add Respond to Webhook Node - Respond to Webhook**  
    - Send HTTP 200 JSON success response with CORS headers  
    - Connect input from Create a post

---

### 5. General Notes & Resources

| Note Content                                                                                                                 | Context or Link                                                                                              |
|------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| Automated LinkedIn Post Designer with N8N Integration. Setup includes webform, HTMLCSSToImage API, OpenAI API, and LinkedIn.  | See sticky note with detailed setup instructions and webform link: https://drive.google.com/file/d/1c2fuxuJXt-eV3qM4aW48H0CVebA1js1j/view?usp=sharing |
| HTML to Image service requires signup at https://htmlcsstoimage.com for free tier (50 images/month).                          | HTML to Image (HCTI) node credentials                                                                       |
| OpenAI API key must be obtained from https://platform.openai.com/settings/ and inserted in the ChatGPT node header.          | ChatGPT - Generate Caption node                                                                              |
| LinkedIn OAuth2 credentials must be created and authorized for the Create a post LinkedIn node.                                | LinkedIn node setup                                                                                           |

---

**Disclaimer:** This documentation is generated exclusively from the provided n8n workflow JSON and reflects its logic and configuration. All data processed is legal and public, respecting content policies.
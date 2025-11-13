Auto-Generate & Post LinkedIn Carousels with Gemini AI and Post Nitro

https://n8nworkflows.xyz/workflows/auto-generate---post-linkedin-carousels-with-gemini-ai-and-post-nitro-7733


# Auto-Generate & Post LinkedIn Carousels with Gemini AI and Post Nitro

---

## 1. Workflow Overview

This workflow automates the creation and posting of LinkedIn carousel posts using AI-generated content and external design and publishing services. It targets content creators, marketers, and professionals who want to automate LinkedIn content creation leveraging the latest tech news, AI content generation, and automated publishing.

The workflow is logically divided into the following blocks:

- **1.1 Triggers:** Two entry points — a scheduled daily trigger at 6:00 AM and a manual form submission trigger for on-demand carousels.
- **1.2 News Retrieval and Parsing:** Automatically fetches TechRadar RSS feed, parses the XML into usable JSON objects.
- **1.3 News Selection by AI:** Uses Google Gemini AI to select the most relevant news item based on profile relevance criteria.
- **1.4 Content Generation:** Generates carousel slides content and post description using Google Gemini AI.
- **1.5 Carousel Creation and PDF Generation:** Sends generated content to Post Nitro to create a designed carousel PDF.
- **1.6 LinkedIn Upload and Posting:** Uploads the PDF to LinkedIn, obtains a document ID, and posts the carousel with the generated description on LinkedIn.

---

## 2. Block-by-Block Analysis

### 2.1 Triggers

**Overview:**  
Provides two ways to start the workflow: automatically every day at 6:00 AM or manually via a web form.

**Nodes Involved:**  
- 6:00 AM Trigger  
- On form submission

**Node Details:**

- **6:00 AM Trigger**  
  - Type: Schedule Trigger  
  - Configuration: Triggers once daily at 6:00 AM server time.  
  - Inputs: None  
  - Outputs: Triggers the News Retrieval block.  
  - Edge Cases: Timezone misalignment, missed triggers if n8n is down.

- **On form submission**  
  - Type: Form Trigger (Webhook)  
  - Configuration: Exposes a webhook at `/create-carousal` path. Form fields: Title (string), Description (textarea), both required.  
  - Inputs: HTTP POST via form submission.  
  - Outputs: Passes form data to content processing block.  
  - Edge Cases: Invalid or incomplete form data, bot submissions filtered out.  
  - Sticky Note: "Triggers - 1. Automated 6:00 AM Trigger 2. Manual Form Trigger"

---

### 2.2 News Retrieval and Parsing

**Overview:**  
Fetches the TechRadar RSS feed and parses it into a structured JSON array of news items.

**Nodes Involved:**  
- TechRadar News  
- Parse HTML

**Node Details:**

- **TechRadar News**  
  - Type: HTTP Request  
  - Configuration: GET request to `https://www.techradar.com/feeds.xml` to retrieve RSS feed.  
  - Outputs: Raw XML data of news feed.  
  - Edge Cases: Network errors, feed format changes, HTTP errors.

- **Parse HTML**  
  - Type: Code (JavaScript)  
  - Configuration: Custom JS to parse the XML RSS feed extracting fields like title, description, link, pubDate, category, media content, etc.  
  - Inputs: Raw RSS XML string from previous node.  
  - Outputs: JSON array of news items with cleaned text fields.  
  - Edge Cases: Parsing errors due to malformed XML or unexpected feed structure.  
  - Sticky Note: "Get News from TechRadar RSS Feed and make it into a readable Object"

---

### 2.3 News Selection by AI

**Overview:**  
Uses Google Gemini AI to select the single most relevant news item tailored to the user's LinkedIn profile interests.

**Nodes Involved:**  
- Gemini Flash 2.5  
- Decide which News to Choose  
- Parse LLM Response

**Node Details:**

- **Gemini Flash 2.5**  
  - Type: Google Gemini AI Chat Model node  
  - Configuration: Sends the news titles list to the AI model for decision making.  
  - Credential: Google Gemini (PaLM) API.  
  - Outputs: AI response with the number of the selected news item.  
  - Edge Cases: API authentication errors, model response delays, invalid responses.  
  - Sticky Note: "Replace Gemini API Key, as well as tell AI about what you post on linkedin"

- **Decide which News to Choose**  
  - Type: Chain LLM (Langchain)  
  - Configuration: Prompt instructs to select the most relevant news number based on specific criteria (automation, AI, personal brand).  
  - Inputs: Parsed news titles JSON.  
  - Outputs: Number indicating selected news item.  
  - Edge Cases: AI returning invalid number or format.

- **Parse LLM Response**  
  - Type: Code  
  - Configuration: Parses the AI’s returned number and extracts the corresponding news item from the parsed news array.  
  - Inputs: AI's selected number and parsed news JSON.  
  - Outputs: Single news item JSON object (title, description, etc.).  
  - Edge Cases: Index out of bounds, parse errors.

- Sticky Note (for this block):  
  "Let AI decide which news resonates with your LinkedIn Profile then format the response"

---

### 2.4 Input Combination and Content Generation

**Overview:**  
Combines inputs from the form submission or AI-selected news, then generates LinkedIn carousel content and post description using Google Gemini AI.

**Nodes Involved:**  
- Parse LLM Response (for form submission input)  
- Get Title and Description  
- Gemini Generate  
- Parse LLM Response1  
- Gemini Generate1

**Node Details:**

- **Parse LLM Response (form input)**  
  - Type: Code  
  - Configuration: Extracts the news item from form submission input if triggered manually.  
  - Outputs: Single news item JSON.

- **Get Title and Description**  
  - Type: Code  
  - Configuration: Creates combinedTitle and combinedDescription by checking multiple possible keys (title/Title, content/Description).  
  - Outputs: Cleaned title and description for AI generation inputs.  
  - Sticky Note: "Combine Form and Automated Inputs"

- **Gemini Generate**  
  - Type: Google Gemini AI (Langchain)  
  - Configuration: Prompts AI to generate LinkedIn carousel slides JSON content (starting_slide, body_slide(s), ending_slide) based on the combined title and description.  
  - Credential: Google Gemini (PaLM) API.  
  - Outputs: Raw AI JSON text of slides.  
  - Edge Cases: JSON parse errors, API quota limits.  
  - Sticky Note: "Replace Gemini API Key"

- **Parse LLM Response1**  
  - Type: Code  
  - Configuration: Cleans and parses AI-generated JSON string into a JSON object for further processing.  
  - Outputs: Parsed slides array.

- **Gemini Generate1**  
  - Type: Google Gemini AI (Langchain)  
  - Configuration: Generates final LinkedIn post description text based on the same title and description inputs.  
  - Outputs: Plain text post description.  
  - Edge Cases: API errors.  
  - Sticky Note: "Generate Final Linkedin Post Description"

---

### 2.5 Carousel Creation and PDF Generation

**Overview:**  
Uses Post Nitro service to create a designed LinkedIn carousel in PDF format from the generated slides JSON.

**Nodes Involved:**  
- ImportSlides embedPost

**Node Details:**

- **ImportSlides embedPost**  
  - Type: Post Nitro AI Node  
  - Configuration: Sends slides JSON to Post Nitro with specific brand and template IDs to generate carousel PDF.  
  - Credentials: Post Nitro API key with brand and template identifiers.  
  - Outputs: URL to generated PDF file.  
  - Edge Cases: API key issues, template or brand ID invalid, network errors.  
  - Sticky Note: "Give your Post Nitro Credentials ( API key + Template ID + Brand ID )"  
  - Sticky Note: "Create Carousal PDF with Post Nitro"

---

### 2.6 LinkedIn Upload and Posting

**Overview:**  
Uploads the generated PDF to LinkedIn's document service, then posts it to the user's LinkedIn feed with the generated description.

**Nodes Involved:**  
- Linkedin User URN  
- Get Upload URL  
- Download PDF  
- Upload PDF  
- Post to LinkedIn

**Node Details:**

- **Linkedin User URN**  
  - Type: HTTP Request  
  - Configuration: GET request to LinkedIn API endpoint `/v2/userinfo` to get authenticated user URN (unique identifier).  
  - Credentials: LinkedIn OAuth2 API credentials.  
  - Outputs: User URN used in upload and posting calls.  
  - Edge Cases: Token expiration, API rate limits.  
  - Sticky Note: "Replace Linkedin API Keys"

- **Get Upload URL**  
  - Type: HTTP Request (POST)  
  - Configuration: Initializes upload session for PDF on LinkedIn documents API. Sends user URN as owner.  
  - Headers: `LinkedIn-Version: 202307`  
  - Outputs: Upload URL and document ID for next steps.  
  - Edge Cases: API version mismatch, authentication errors.

- **Download PDF**  
  - Type: HTTP Request  
  - Configuration: Downloads generated PDF file from Post Nitro URL.  
  - Response Format: Binary file, output stored as `my.pdf`.  
  - Edge Cases: Network errors, file not found.

- **Upload PDF**  
  - Type: HTTP Request (PUT)  
  - Configuration: Uploads binary PDF data to LinkedIn using upload URL from previous step.  
  - Headers: `Content-Type: application/pdf`  
  - Uses binary data field `my.pdf`.  
  - Edge Cases: Upload failures, content length limits.

- **Post to LinkedIn**  
  - Type: HTTP Request (POST)  
  - Configuration: Posts new LinkedIn post with content referencing the uploaded document ID and generated post description text.  
  - Headers: `LinkedIn-Version: 202408`, `Content-Type: application/json`  
  - Payload includes author URN, commentary text, media content with document ID, visibility set to public.  
  - Edge Cases: Post rejection, API rate limits, invalid document references.  
  - Sticky Note: "Post to LinkedIn"

---

## 3. Summary Table

| Node Name               | Node Type                                    | Functional Role                             | Input Node(s)              | Output Node(s)          | Sticky Note                                                                                                           |
|-------------------------|----------------------------------------------|---------------------------------------------|----------------------------|-------------------------|-----------------------------------------------------------------------------------------------------------------------|
| 6:00 AM Trigger          | Schedule Trigger                             | Automated daily trigger                      | —                          | TechRadar News           | Triggers 1. Automated 6:00 AM Trigger 2. Manual Form Trigger                                                           |
| On form submission       | Form Trigger (Webhook)                       | Manual trigger with form input               | —                          | Get Title and Description | Triggers 1. Automated 6:00 AM Trigger 2. Manual Form Trigger                                                           |
| TechRadar News           | HTTP Request                                | Fetch TechRadar RSS feed                      | 6:00 AM Trigger            | Parse HTML               | Get News from TechRadar RSS Feed and make it into a readable Object                                                    |
| Parse HTML              | Code (JavaScript)                            | Parse RSS XML into JSON array                 | TechRadar News             | Decide which News to Choose | Get News from TechRadar RSS Feed and make it into a readable Object                                                    |
| Gemini Flash 2.5         | Google Gemini AI Chat Model                  | AI model to select most relevant news item   | Parse HTML                 | Decide which News to Choose | Replace Gemini API Key, as well as tell AI about what you post on linkedin                                              |
| Decide which News to Choose | Chain LLM (Langchain)                     | Selects most relevant news number             | Parse HTML, Gemini Flash 2.5 | Parse LLM Response       | Let AI decide which news resonates with your LinkedIn Profile then format the response                                 |
| Parse LLM Response       | Code                                         | Extracts selected news item by index          | Decide which News to Choose | Get Title and Description | Let AI decide which news resonates with your LinkedIn Profile then format the response                                 |
| Parse LLM Response (form) | Code                                        | Extracts news item from form input            | On form submission          | Get Title and Description | Let AI decide which news resonates with your LinkedIn Profile then format the response                                 |
| Get Title and Description | Code                                         | Combines and cleans title and description     | Parse LLM Response, Parse LLM Response (form) | Gemini Generate            | Combine Form and Automated Inputs                                                                                      |
| Gemini Generate          | Google Gemini AI (Langchain)                 | Generates LinkedIn carousel slides JSON       | Get Title and Description   | Parse LLM Response1       | Replace Gemini API Key                                                                                                  |
| Parse LLM Response1      | Code                                         | Parses AI-generated JSON slides content       | Gemini Generate            | ImportSlides embedPost    |                                                                                                                       |
| ImportSlides embedPost   | Post Nitro AI Node                           | Creates carousel PDF from slides JSON         | Parse LLM Response1         | Linkedin User URN         | Give your Post Nitro Credentials ( API key + Template ID + Brand ID ) Create Carousal PDF with Post Nitro              |
| Linkedin User URN        | HTTP Request                                | Gets LinkedIn user URN for upload/posting     | ImportSlides embedPost      | Get Upload URL            | Replace Linkedin API Keys                                                                                              |
| Get Upload URL           | HTTP Request (POST)                         | Initializes LinkedIn PDF upload session       | Linkedin User URN           | Download PDF              | Replace Linkedin API Keys                                                                                              |
| Download PDF             | HTTP Request                                | Downloads PDF file from Post Nitro             | Get Upload URL              | Upload PDF                | Upload PDF to LinkedIn                                                                                                |
| Upload PDF               | HTTP Request (PUT)                          | Uploads PDF binary data to LinkedIn            | Download PDF                | Gemini Generate1          | Upload PDF to LinkedIn                                                                                                |
| Gemini Generate1         | Google Gemini AI (Langchain)                 | Generates LinkedIn post description text       | Upload PDF                  | Post to LinkedIn          | Replace Gemini API Key; Generate Final Linkedin Post Description                                                       |
| Post to LinkedIn         | HTTP Request (POST)                         | Creates LinkedIn post with PDF carousel        | Gemini Generate1            | —                         | Post to LinkedIn                                                                                                      |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Triggers**
   - Create a **Schedule Trigger** node named `6:00 AM Trigger`.
     - Set to trigger daily at 6:00 AM server time.
   - Create a **Form Trigger** node named `On form submission`.
     - Path: `create-carousal`
     - Form fields:  
       - Title (string, required)  
       - Description (textarea, required)  
     - Enable "Ignore Bots" option.

2. **Fetch and Parse News**
   - Add **HTTP Request** node `TechRadar News`.
     - Method: GET  
     - URL: `https://www.techradar.com/feeds.xml`
     - Connect output of `6:00 AM Trigger` to this node.
   - Add **Code** node `Parse HTML`.
     - Implement JavaScript to parse RSS XML into a JSON array extracting fields like title, description, link, pubDate, etc.  
     - Connect output of `TechRadar News` here.

3. **AI News Selection**
   - Add **Google Gemini AI Chat Model** node `Gemini Flash 2.5`.
     - Credential: Google Gemini (PaLM) API  
     - Connect output of `Parse HTML` here.
   - Add **Chain LLM** node `Decide which News to Choose`.
     - Use prompt to select the most relevant news number based on provided criteria and news titles list.  
     - Connect `Gemini Flash 2.5` output here.
   - Add **Code** node `Parse LLM Response`.
     - Parse AI's returned number and extract corresponding news item from parsed news array.  
     - Connect `Decide which News to Choose` output here.

4. **Handle Manual Form Submission Input**
   - Add **Code** node `Parse LLM Response (form)`.
     - Extracts news item from form submission input.  
     - Connect output of `On form submission` here.

5. **Combine Inputs**
   - Add **Code** node `Get Title and Description`.
     - Combine title and description fields from either AI-selected news or form submission.  
     - Connect outputs of both `Parse LLM Response` and `Parse LLM Response (form)` here (merge using a merge node or conditional logic to choose source).

6. **Generate Carousel Content**
   - Add **Google Gemini AI (Langchain)** node `Gemini Generate`.
     - Credential: Google Gemini (PaLM) API  
     - Prompt: Generate LinkedIn carousel slides JSON with a strict schema (starting_slide, body_slide(s), ending_slide).  
     - Use combined title and description as input.  
     - Connect output of `Get Title and Description` here.
   - Add **Code** node `Parse LLM Response1`.
     - Parse AI-generated JSON string into JSON object.  
     - Connect `Gemini Generate` output here.

7. **Create Carousel PDF**
   - Add **Post Nitro** node `ImportSlides embedPost`.
     - Provide:  
       - Brand ID  
       - Template ID  
       - Slides JSON (from `Parse LLM Response1`)  
     - Credentials: Post Nitro API key  
     - Connect output of `Parse LLM Response1` here.

8. **LinkedIn Upload and Post**
   - Add **HTTP Request** node `Linkedin User URN`.
     - GET `https://api.linkedin.com/v2/userinfo`  
     - Credentials: LinkedIn OAuth2  
     - Connect output of `ImportSlides embedPost` here.
   - Add **HTTP Request** node `Get Upload URL`.
     - POST `https://api.linkedin.com/rest/documents?action=initializeUpload`  
     - Body: JSON with owner URN from previous node  
     - Header: `LinkedIn-Version: 202307`  
     - Credentials: LinkedIn OAuth2  
     - Connect output of `Linkedin User URN` here.
   - Add **HTTP Request** node `Download PDF`.
     - GET PDF file URL from Post Nitro output  
     - Response: Binary file stored as `my.pdf`  
     - Connect output of `Get Upload URL` here.
   - Add **HTTP Request** node `Upload PDF`.
     - PUT to upload URL from `Get Upload URL` output  
     - Content-Type: application/pdf  
     - Send binary data `my.pdf`  
     - Credentials: LinkedIn OAuth2  
     - Connect output of `Download PDF` here.
   - Add **Google Gemini AI (Langchain)** node `Gemini Generate1`.
     - Credential: Google Gemini (PaLM) API  
     - Prompt: Generate final LinkedIn post description text based on combined title and description.  
     - Connect output of `Upload PDF` here.
   - Add **HTTP Request** node `Post to LinkedIn`.
     - POST `https://api.linkedin.com/rest/posts`  
     - Body: JSON including author URN, commentary text from `Gemini Generate1`, visibility, and media referencing uploaded document ID.  
     - Headers: `LinkedIn-Version: 202408`, `Content-Type: application/json`  
     - Credentials: LinkedIn OAuth2  
     - Connect output of `Gemini Generate1` here.

9. **Final Steps**
   - Ensure error handling and node execution order is configured to stop or notify on failures in critical nodes (e.g., AI calls, LinkedIn uploads).
   - Set workflow to active.
   - Test manually using form submission and verify automatic 6:00 AM runs.

---

## 5. General Notes & Resources

| Note Content                                                                                                                       | Context or Link                                                                                              |
|-----------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| Use case: LinkedIn Content Creation, specifically carousel posts automated end-to-end with AI and Post Nitro.                      | Sticky Note10: Workflow overview and instructions.                                                          |
| Requires Post Nitro community node `@postnitro/n8n-nodes-postnitro-ai`.                                                            | Post Nitro Docs: https://postnitro.ai/docs/embed/obtaining-an-api-key                                        |
| Requires Google Gemini (PaLM) API credentials for AI content generation.                                                           | Google Gemini Docs: https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.lmchatgooglegemini/ |
| Requires LinkedIn API OAuth2 credentials for uploading and posting content.                                                        | LinkedIn API Docs: https://docs.n8n.io/integrations/builtin/credentials/linkedin                              |
| Replace all placeholder API keys and IDs for Gemini, Post Nitro, and LinkedIn before running.                                       | Sticky Notes 1, 11, 12, 13, 14, 15                                                                           |
| For help or questions, contact author on LinkedIn: https://www.linkedin.com/in/shayan-khan20/                                      | Sticky Note10                                                                                                |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.

---
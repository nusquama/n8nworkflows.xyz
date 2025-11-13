Automate RSS News to Telegram with GPT AI & Image Watermarking

https://n8nworkflows.xyz/workflows/automate-rss-news-to-telegram-with-gpt-ai---image-watermarking-9476


# Automate RSS News to Telegram with GPT AI & Image Watermarking

### 1. Workflow Overview

This workflow automates the process of monitoring an RSS news feed, enhancing article content using AI, watermarking article images, and publishing the curated posts directly to a Telegram channel. It is designed to keep Telegram channels updated with fresh, well-formatted news items while ensuring no duplicate content is posted. The workflow is structured into three main logical blocks:

- **1.1 Detect & Filter New Articles**: Monitors the RSS feed, checks if an article link has already been processed via a Google Sheet, and filters duplicates to avoid reposting.
- **1.2 Enhance & Prepare Content**: Uses an AI agent (OpenAI GPT-4.1-mini) to rewrite and format the article text for better clarity, engagement, and Telegram-friendly formatting.
- **1.3 Image Processing & Publishing**: Fetches the article's main image, adds a watermark, and publishes the final image along with the AI-enhanced text to a Telegram channel.

---

### 2. Block-by-Block Analysis

#### 1.1 Detect & Filter New Articles

**Overview:**  
This initial block triggers on new RSS feed items, verifies if the article URL was already processed by consulting a Google Sheet, and filters out duplicates to ensure only new articles proceed.

**Nodes Involved:**  
- RSS Feed Trigger  
- Check Processed Links (Google Sheets)  
- Check: Is Link New? (IF)  
- Skip: Link Already Processed (NoOp)  
- Update Processed Links (Google Sheets)

**Node Details:**  

- **RSS Feed Trigger**  
  - Type: RSS Feed Read Trigger  
  - Role: Polls the RSS feed URL (`https://www.kinonews.ru/rss/`) every 15 minutes for new articles.  
  - Config: Poll interval set to every 15 minutes; uses RSS feed URL as source.  
  - Inputs: None (trigger node)  
  - Outputs: Emits new feed items with metadata including title, link, content, and enclosure URLs.  
  - Failure Modes: Feed inaccessible or malformed RSS, network timeout, no new items.  
  - Notes: Core entry point for the workflow.  

- **Check Processed Links**  
  - Type: Google Sheets (Read)  
  - Role: Reads a specific sheet (`kinonews.ru`) in Google Sheets document to get URLs of already processed articles.  
  - Config: Uses Google Sheets OAuth2 credentials; document ID and sheet name predefined; reads all rows to compare URLs.  
  - Inputs: Output of RSS Feed Trigger  
  - Outputs: List of URLs from the sheet for comparison.  
  - Failure Modes: Google Sheets API auth errors, document access issues, quota limits.  

- **Check: Is Link New?**  
  - Type: IF node  
  - Role: Compares the current RSS item's URL against URLs retrieved from Google Sheets to determine if the article is new or duplicate.  
  - Config: Checks equality between the current RSS link and each URL in the sheet (strict, case-sensitive).  
  - Inputs: Output of Check Processed Links and RSS Feed Trigger item for comparison.  
  - Outputs:  
    - True branch: Link already processed  
    - False branch: New link to process  
  - Failure Modes: Expression evaluation errors, empty or malformed data.  

- **Skip: Link Already Processed**  
  - Type: NoOp (No operation)  
  - Role: Terminates processing for duplicate links cleanly without error.  
  - Inputs: True branch of IF node  
  - Outputs: None (end branch)  
  - Failure Modes: None expected.  

- **Update Processed Links**  
  - Type: Google Sheets (Update)  
  - Role: Adds or updates the Google Sheet record for the newly processed URL to prevent future duplication.  
  - Config: Maps RSS item URL to the corresponding row number; uses same document and sheet as reading node; updates URL in sheet.  
  - Inputs: False branch of IF node (new links)  
  - Outputs: Passes updated data downstream for further processing.  
  - Failure Modes: Google Sheets auth errors or quota limits, update conflicts.  

---

#### 1.2 Enhance & Prepare Content

**Overview:**  
Uses an AI agent to rewrite and format the article text for clarity, engagement, and Telegram-friendly markdown formatting, including character limit enforcement and emoji use.

**Nodes Involved:**  
- Customize Article Text (Langchain Agent)  
- OpenAI Chat Model (Langchain LM Chat OpenAI)

**Node Details:**  

- **Customize Article Text**  
  - Type: Langchain Agent node  
  - Role: Sends the article content to an AI agent that formats and condenses the text with Telegram markdown and emojis without altering meaning.  
  - Config:  
    - System message instructs the agent to:  
      * Format with **bold**, *italic*, and emojis  
      * Condense text if over 700 characters  
      * Return a ready-to-publish text block  
    - Inputs: RSS Feed Trigger fields `title` and `content` interpolated in prompt.  
  - Inputs: Output of Update Processed Links node  
  - Outputs: AI-enhanced formatted text for publication.  
  - Failure Modes: OpenAI API errors, prompt misinterpretation, timeouts, token limits.  

- **OpenAI Chat Model**  
  - Type: Langchain LM Chat OpenAI  
  - Role: Provides the underlying OpenAI GPT-4.1-mini model for the AI agent.  
  - Config: Model set to "gpt-4.1-mini" with default options; no special parameters.  
  - Inputs: Invoked internally by the Customize Article Text node.  
  - Outputs: AI-generated text responses.  
  - Failure Modes: API quota limits, network timeouts, authentication errors.  

---

#### 1.3 Image Processing & Publishing

**Overview:**  
Fetches the article’s main image, applies a watermark, then publishes the watermarked image along with the AI-enhanced text to a Telegram channel.

**Nodes Involved:**  
- Fetch Article Image (HTTP Request)  
- Add Watermark to Image (Edit Image)  
- Publish to Telegram Channel (Telegram)

**Node Details:**  

- **Fetch Article Image**  
  - Type: HTTP Request  
  - Role: Downloads the article’s main image from the URL provided in the RSS feed enclosure field.  
  - Config: Simple GET request to the image URL (`enclosure.url`).  
  - Inputs: Output of Customize Article Text node  
  - Outputs: Binary image data for the next node.  
  - Failure Modes: Image URL inaccessible, HTTP errors, timeouts, invalid image format.  

- **Add Watermark to Image**  
  - Type: Edit Image  
  - Role: Adds a textual watermark "AI.NEWS" to the downloaded image with specified font and styling.  
  - Config:  
    - Text: "AI.NEWS"  
    - Font: Verdana Bold Italic (path `/usr/share/fonts/truetype/msttcorefonts/Verdana_Bold_Italic.ttf`)  
    - Font size: 48  
    - Font color: #F3ECEC (light gray)  
    - Position Y: 150 pixels from top  
    - Operation: Text overlay  
  - Inputs: Binary data from Fetch Article Image  
  - Outputs: Watermarked image data for publishing.  
  - Failure Modes: Font file missing, image format unsupported, memory limits.  

- **Publish to Telegram Channel**  
  - Type: Telegram node  
  - Role: Sends the watermarked image and AI-enhanced article text to a Telegram channel via a configured bot.  
  - Config:  
    - Operation: sendPhoto  
    - Uses binaryData for image  
    - Caption set to AI-enhanced text (`$json.output`)  
    - Parse mode: Markdown for Telegram formatting  
    - Credentials: Telegram Bot API token configured under "Workflow Bot maniac"  
  - Inputs: Output from Add Watermark to Image and Customize Article Text (caption)  
  - Outputs: Confirmation or error message from Telegram API.  
  - Failure Modes: Telegram API token invalid, bot permissions lacking, network errors, image size limits.  

---

### 3. Summary Table

| Node Name               | Node Type                         | Functional Role                            | Input Node(s)             | Output Node(s)                         | Sticky Note                                                                                   |
|-------------------------|----------------------------------|--------------------------------------------|---------------------------|---------------------------------------|-----------------------------------------------------------------------------------------------|
| RSS Feed Trigger         | RSS Feed Read Trigger             | Entry point: monitors RSS feed every 15 min | None                      | Check Processed Links                  | **Automated Telegram RSS Publisher** posts new articles with AI & watermarking.               |
| Check Processed Links    | Google Sheets                    | Reads sheet to get processed URLs          | RSS Feed Trigger          | Check: Is Link New?                    | **1. Detect & Filter New Articles** Monitors RSS and skips already processed links.           |
| Check: Is Link New?      | IF                              | Verifies if article URL is new or duplicate | Check Processed Links     | Skip: Link Already Processed (true) / Update Processed Links (false) |                                                                                               |
| Skip: Link Already Processed | NoOp                          | Ends processing for duplicates              | Check: Is Link New? (true)| None                                  |                                                                                               |
| Update Processed Links   | Google Sheets                   | Updates sheet with new article URL          | Check: Is Link New? (false)| Customize Article Text                |                                                                                               |
| Customize Article Text   | Langchain Agent                 | AI rewrites and formats article text        | Update Processed Links    | Fetch Article Image                   | **2️. Enhance & Prepare Content** AI polishes text for Telegram publishing.                    |
| OpenAI Chat Model        | Langchain LM Chat OpenAI         | Provides GPT-4.1-mini AI model              | Internal to Customize Article Text | Customize Article Text (AI output) |                                                                                               |
| Fetch Article Image      | HTTP Request                    | Downloads article main image                 | Customize Article Text    | Add Watermark to Image                | **3️. Publish to Telegram** Fetches image, watermarks it, and sends to Telegram channel.       |
| Add Watermark to Image   | Edit Image                      | Adds "AI.NEWS" watermark on image           | Fetch Article Image       | Publish to Telegram Channel           |                                                                                               |
| Publish to Telegram Channel | Telegram                      | Sends photo and text to Telegram channel    | Add Watermark to Image    | None                                  |                                                                                               |
| Sticky Note              | Sticky Note                     | Workflow description and overview           | None                      | None                                  | **Automated Telegram RSS Publisher** overview and requirements.                               |
| Sticky Note1             | Sticky Note                     | Describes detection and filtering block     | None                      | None                                  | **1. Detect & Filter New Articles** details.                                                 |
| Sticky Note2             | Sticky Note                     | Describes AI enhancement block               | None                      | None                                  | **2️. Enhance & Prepare Content** AI text rewriting notes.                                   |
| Sticky Note3             | Sticky Note                     | Describes publishing block                    | None                      | None                                  | **3️. Publish to Telegram** image watermarking and publishing explanation.                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create RSS Feed Trigger node:**  
   - Type: RSS Feed Read Trigger  
   - Set feed URL: `https://www.kinonews.ru/rss/`  
   - Set polling interval: every 15 minutes  

2. **Add Google Sheets node for reading processed links:**  
   - Type: Google Sheets (Read)  
   - Credentials: Use Google Sheets OAuth2 credentials  
   - Document ID: `1MhbaPSqLRXsJfeR-2JlvrIyFxvprgkmzIE5bib7-7MQ`  
   - Sheet Name: `kinonews.ru` (Sheet ID 538399553)  
   - Operation: Read all rows to fetch stored URLs  

3. **Connect RSS Feed Trigger output to Check Processed Links input.**

4. **Add IF node to check if RSS link is new:**  
   - Condition: Check if any URL from sheet equals current RSS item link (`$json.url == $('RSS Feed Trigger').item.json.link`)  
   - Case sensitive, strict type validation  

5. **Add NoOp node labeled "Skip: Link Already Processed":**  
   - Connect IF node "true" branch here to ignore duplicates  

6. **Add Google Sheets node for updating processed links:**  
   - Same credentials and document as reading node  
   - Operation: Update  
   - Mapping: Map `url` field with current RSS link; use `row_number` to target correct row for update  

7. **Connect IF node "false" branch (new links) to Update Processed Links node.**

8. **Add Langchain Agent node "Customize Article Text":**  
   - System Message Prompt:  
     ```
     You are a news editor for a Telegram channel. Your responsibilities include only adding beautiful formatting to the news text—without changing the actual text. If the original text exceeds 700 characters, condense it in a way that preserves the meaning and ensures the text never exceeds 700 characters.

     Use Telegram's built-in formatting options, such as **Bold** and *Italic*, as well as appropriate emojis.

     Create a beautifully formatted version of the news titled:
     "{{ $('RSS Feed Trigger').item.json.title }}"

     ---
     {{ $('RSS Feed Trigger').item.json.content }}

     ---
     The response should be a ready-to-publish text.
     ```  
   - Inputs: Pass data from Update Processed Links node (including title & content)  
   - Prompt Type: Define  

9. **Configure OpenAI Chat Model node:**  
   - Model: `gpt-4.1-mini`  
   - Connect as AI language model for the Langchain Agent node  

10. **Connect Update Processed Links node output to Customize Article Text node input.**

11. **Add HTTP Request node "Fetch Article Image":**  
    - Method: GET  
    - URL: Use expression to get image URL from RSS item enclosure (`$('RSS Feed Trigger').item.json.enclosure.url`)  

12. **Connect Customize Article Text node output to Fetch Article Image node input.**

13. **Add Edit Image node "Add Watermark to Image":**  
    - Operation: Text overlay  
    - Text: "AI.NEWS"  
    - Font: Verdana_Bold_Italic.ttf (path `/usr/share/fonts/truetype/msttcorefonts/Verdana_Bold_Italic.ttf`)  
    - Font size: 48  
    - Font color: #F3ECEC  
    - Position Y: 150  
    - Input data property: image binary data from HTTP Request node  

14. **Connect Fetch Article Image node to Add Watermark to Image node.**

15. **Add Telegram node "Publish to Telegram Channel":**  
    - Operation: sendPhoto  
    - Use Binary Data: true (to send image)  
    - Caption: Use AI-enhanced text output (`$json.output`) from Customize Article Text node  
    - Parse mode: Markdown  
    - Credentials: Configure Telegram Bot API credentials (e.g., "Workflow Bot maniac")  

16. **Connect Add Watermark to Image node output to Publish to Telegram Channel node input.**

17. **Optionally add sticky notes for documentation and overview at appropriate positions.**

18. **Activate the workflow when ready.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                                                                          |
|------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| The workflow requires a Google Sheet to track processed article URLs to prevent duplicate postings.                               | Google Sheets document ID: `1MhbaPSqLRXsJfeR-2JlvrIyFxvprgkmzIE5bib7-7MQ`                              |
| Telegram Bot token with permission to post to your channel must be configured under credentials.                                  | Ensure bot has permission to send photos and messages to the channel.                                  |
| AI text rewriting uses OpenAI GPT-4.1-mini model via Langchain integration.                                                        | Model may require API key and quota management.                                                        |
| Watermark font file `Verdana_Bold_Italic.ttf` must be accessible on the n8n host system at specified path.                        | Adjust font path or font if the file is missing or incompatible.                                       |
| For best results, ensure RSS feed URLs and enclosure URLs are valid and accessible.                                                | RSS feed: `https://www.kinonews.ru/rss/`                                                               |
| Workflow logic carefully handles duplicates, so Google Sheet integrity is essential.                                              | Avoid concurrent edits or race conditions on the sheet.                                               |
| Sticky notes in workflow provide detailed explanations of each logic block and node purpose.                                      | Useful for onboarding new developers or auditing.                                                     |

---

**Disclaimer:** The text provided is exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and publicly available.
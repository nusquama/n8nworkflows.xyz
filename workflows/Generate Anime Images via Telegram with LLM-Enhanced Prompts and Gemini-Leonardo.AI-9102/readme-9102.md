Generate Anime Images via Telegram with LLM-Enhanced Prompts and Gemini/Leonardo.AI

https://n8nworkflows.xyz/workflows/generate-anime-images-via-telegram-with-llm-enhanced-prompts-and-gemini-leonardo-ai-9102


# Generate Anime Images via Telegram with LLM-Enhanced Prompts and Gemini/Leonardo.AI

### 1. Workflow Overview

This workflow enables users to generate high-quality anime-style images directly via Telegram messages. It accepts simple text prompts from a Telegram chat, then uses a Large Language Model (LLM) to enrich these prompts into detailed, vivid descriptions optimized for AI image generation. The enhanced prompts are then sent to an AI image generation service—by default Google Gemini's API or optionally Leonardo.AI—to create anime images. Finally, the generated images are converted from base64 to files and sent back to the user on Telegram.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receiving user prompts from Telegram.
- **1.2 Prompt Enhancement (LLM Processing):** Using an LLM to transform simple user input into rich, multi-part anime prompts.
- **1.3 Prompt Preparation:** Parsing and splitting the enriched prompts for batch processing.
- **1.4 Image Generation:** Calling the AI image generation API (Gemini by default, Leonardo.AI optionally) to generate images from the prompts.
- **1.5 Post-Processing and Delivery:** Converting API image data to files and sending images back to Telegram.
- **1.6 Auxiliary & Configuration Notes:** Sticky notes provide setup instructions, tips, and optional configurations.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens for incoming Telegram messages from users. It captures the user’s raw text prompt and initializes the workflow by setting the number of images to generate (default: 4).

- **Nodes Involved:**  
  - Telegram Trigger  
  - Image-count

- **Node Details:**

  - **Telegram Trigger**  
    - Type: Trigger node for Telegram messaging platform.  
    - Configuration: Listens to all incoming messages (`updates`: ["message"]) with no chat ID restriction by default (empty `chatIds` field).  
    - Input: Telegram messages.  
    - Output: Message JSON containing user text.  
    - Credentials: Requires Telegram Bot API token (configured with "LXirein Bot (SMS)").  
    - Edge Cases:  
      - Missing or incorrect Telegram credentials will cause auth failure.  
      - No chat ID restriction could allow messages from any user (can be restricted by adding chat IDs).  
      - Message types other than text may be ignored.

  - **Image-count**  
    - Type: Set node.  
    - Configuration: Extracts the raw text prompt from incoming Telegram message (`User_Prompt`) and sets default number of images to generate (`num_of_Images` = 4).  
    - Input: Telegram Trigger output.  
    - Output: JSON with prompt and image count.  
    - Edge Cases:  
      - If message text is empty or malformed, prompt will be empty.  
      - Number of images is hardcoded as 4 but can be made dynamic.

---

#### 2.2 Prompt Enhancement (LLM Processing)

- **Overview:**  
  Enhances the simple user prompt into multiple detailed and artistically rich anime-style prompts using a Large Language Model (LLM). The output is a structured array of enriched prompt descriptions.

- **Nodes Involved:**  
  - Prompt generator (USER_Inspired)  
  - Structured Output Parser  
  - DeepSeek from OpenRouter

- **Node Details:**

  - **Prompt generator (USER_Inspired)**  
    - Type: LangChain LLM Chain node.  
    - Configuration:  
      - Input text template uses the user prompt and desired number of images to generate multiple detailed anime prompts.  
      - LLM acts as an expert anime artist and prompt generator.  
      - Output is structured JSON array with objects containing a detailed `description` field.  
    - Input: JSON with `User_Prompt` and `num_of_Images`.  
    - Output: Array of refined prompts.  
    - Expressions: Uses expressions to dynamically insert user input and number of images.  
    - Edge Cases:  
      - LLM API failures (rate limit, timeout) will stop workflow.  
      - Malformed user input may affect prompt quality.  
      - Output parser relies on strict JSON schema; deviations may cause parsing errors.

  - **Structured Output Parser**  
    - Type: LangChain structured output parser node.  
    - Configuration: Parses LLM output into a strict JSON array schema to ensure predictable downstream processing.  
    - Input: Raw LLM text output.  
    - Output: Parsed JSON array of prompt objects with `description` strings.  
    - Edge Cases:  
      - If LLM output is not valid JSON or does not conform, parsing fails.

  - **DeepSeek from OpenRouter**  
    - Type: LangChain Chat LLM node.  
    - Configuration: Uses `deepseek/deepseek-chat-v3-0324:free` model via OpenRouter API to run the prompt enhancement LLM.  
    - Credentials: OpenRouter API key required.  
    - Input: Prompt text from "Image-count".  
    - Output: Raw LLM response to be parsed by "Structured Output Parser".  
    - Edge Cases:  
      - API key or connectivity issues cause failures.  
      - Model rate limits or downtime affect prompt generation.

---

#### 2.3 Prompt Preparation

- **Overview:**  
  Splits the array of detailed prompts into individual items for batch processing in parallel image generation.

- **Nodes Involved:**  
  - Split Out Prompts  
  - Loop Over Prompts

- **Node Details:**

  - **Split Out Prompts**  
    - Type: SplitOut node.  
    - Configuration: Splits the `output` field (array of prompts) into multiple items.  
    - Input: Parsed prompt array from "Prompt generator (USER_Inspired)".  
    - Output: Individual prompt JSON objects.  
    - Edge Cases:  
      - Empty or missing array input will halt further processing.

  - **Loop Over Prompts**  
    - Type: SplitInBatches node.  
    - Configuration: Processes each prompt individually or in batches (default batch size 1).  
    - Input: Individual prompt objects from "Split Out Prompts".  
    - Output: Each prompt fed into image generation and Telegram sending nodes.  
    - Edge Cases:  
      - Large batch sizes risk rate limiting or timeouts in downstream nodes.

---

#### 2.4 Image Generation

- **Overview:**  
  Sends each detailed prompt to an AI image generation API to produce anime-style images. The default service is Google Gemini (part of PaLM API) offering free usage for 90 days. Optionally, the Leonardo.AI API node can replace Gemini for potentially superior quality.

- **Nodes Involved:**  
  - HTTP- Gemini  
  - HTTP- Leonardo AI (optional)  
  - Set base64

- **Node Details:**

  - **HTTP- Gemini**  
    - Type: HTTP Request node.  
    - Configuration:  
      - POST request to Gemini 2.0 image generation endpoint.  
      - Request body includes the detailed prompt as JSON text, requesting both TEXT and IMAGE response modalities.  
      - Authentication via predefined Google Palm API credential.  
    - Input: Prompt description JSON from "Loop Over Prompts".  
    - Output: JSON including base64-encoded image data nested inside response structure.  
    - Edge Cases:  
      - API key or quota issues cause auth or rate-limit errors.  
      - Unexpected response structure may break downstream processing.

  - **HTTP- Leonardo AI**  
    - Type: HTTP Request node (alternative to Gemini).  
    - Configuration:  
      - POST to Leonardo.AI generation API with parameters for image dimensions, style, guidance scale, and prompt.  
      - Uses HTTP Header Auth credential with Leonardo API key.  
      - `promptMagic` and other style flags enabled for anime style.  
    - Input: Prompt description JSON.  
    - Output: Leonardo API response with generated images (not fully detailed in workflow).  
    - Edge Cases:  
      - Requires valid Leonardo API key and paid subscription.  
      - API limits, errors, or changed API spec can cause failures.

  - **Set base64**  
    - Type: Set node.  
    - Configuration: Extracts base64 image data from Gemini response JSON, checking nested fields for image data.  
    - Input: Gemini HTTP response JSON.  
    - Output: JSON field `base64` containing image binary data encoded as base64 string.  
    - Edge Cases:  
      - Missing or malformed image data will result in empty or invalid base64.

---

#### 2.5 Post-Processing and Delivery

- **Overview:**  
  Converts the base64 image data to a binary file and sends the image back to the Telegram user. Includes a wait node to avoid hitting rate limits.

- **Nodes Involved:**  
  - Convert to File  
  - Wait 10s  
  - Send pics via Telegram

- **Node Details:**

  - **Convert to File**  
    - Type: ConvertToFile node.  
    - Configuration: Converts the `base64` string field into binary file data for sending over Telegram.  
    - Input: JSON with base64 image data.  
    - Output: Binary file attached to item.  
    - Edge Cases:  
      - Invalid base64 strings cause conversion errors.

  - **Wait 10s**  
    - Type: Wait node.  
    - Configuration: Pauses workflow for 10 seconds between sending images to avoid API rate limits or flooding Telegram.  
    - Input: Binary file item.  
    - Output: Delayed item.  
    - Edge Cases:  
      - Long delays slow user experience but protect from rate limits.

  - **Send pics via Telegram**  
    - Type: Telegram node (message sender).  
    - Configuration: Sends photo to user’s chat ID with binary data from previous node.  
    - Input: Binary file.  
    - Credentials: Same Telegram Bot API as trigger node.  
    - Edge Cases:  
      - Incorrect chat ID or revoked bot permissions cause send failures.  
      - Large files or too many rapid requests may cause Telegram API rate limits.

---

#### 2.6 Auxiliary & Configuration Notes

- Several Sticky Note nodes provide guidance:

  - Setup instructions for Telegram bot, API credentials, and image generation options.
  - Tips for prompt crafting and API usage.
  - Documentation about switching from Gemini to Leonardo.AI.
  - Warnings about API keys privacy, usage constraints, and NSFW content avoidance.

---

### 3. Summary Table

| Node Name                  | Node Type                          | Functional Role                      | Input Node(s)                  | Output Node(s)                  | Sticky Note                                                  |
|----------------------------|----------------------------------|------------------------------------|-------------------------------|-------------------------------|--------------------------------------------------------------|
| Sticky Note                | Sticky Note                      | Setup info for Gemini image gen    |                               |                               | # Image generation with Gemini [Free for 90-days with a GCP acc] |
| Telegram Trigger           | Telegram Trigger                 | Receive user prompt via Telegram   |                               | Image-count                   | ## Configure Me! Paste your own chatID in the "Restrict to Chat IDs" field |
| Image-count                | Set                             | Extract prompt and set image count | Telegram Trigger              | Prompt generator (USER_Inspired) | ## Generates 4 images for an image query                      |
| Prompt generator (USER_Inspired) | LangChain Chain LLM             | Enhance prompt into detailed prompts | Image-count                  | Split Out Prompts             |                                                              |
| Structured Output Parser   | LangChain Structured Output Parser | Parse LLM output to JSON array     | Prompt generator (USER_Inspired) | Split Out Prompts             |                                                              |
| DeepSeek from OpenRouter   | LangChain Chat LLM (OpenRouter) | LLM model provider for prompt gen  | Image-count                   | Prompt generator (USER_Inspired) |                                                              |
| Split Out Prompts          | Split Out                       | Split array of prompts into items  | Prompt generator (USER_Inspired) | Loop Over Prompts             |                                                              |
| Loop Over Prompts          | Split In Batches                | Process each prompt individually   | Split Out Prompts             | HTTP- Gemini, Send pics via Telegram |                                                              |
| HTTP- Gemini               | HTTP Request                   | Generate image via Gemini API      | Loop Over Prompts             | Set base64                   | ## Using gemini model to generate image                      |
| Set base64                 | Set                             | Extract base64 image data          | HTTP- Gemini                  | Convert to File              |                                                              |
| Convert to File            | Convert To File                 | Convert base64 string to file      | Set base64                   | Wait 10s                    |                                                              |
| Wait 10s                   | Wait                           | Pause to avoid rate limits         | Convert to File              | Loop Over Prompts            | ## Not necessary Used to avoid rate limits                   |
| Send pics via Telegram     | Telegram                       | Send generated image to user       | Loop Over Prompts             |                               | ## Configure Me! Paste your own chatID in the "ChatID" field |
| HTTP- Leonardo AI          | HTTP Request                   | Optional: Generate image via Leonardo.AI | Loop Over Prompts          | (not connected in default flow) | ## Using Leonardo.AI to generate image. Replace `HTTP- Gemini` node with this node. Create new header auth credential with your leonardo.ai API key. |
| Sticky Note1               | Sticky Note                    | Note about Gemini node usage       |                               |                               | ## Using gemini model to generate image                      |
| Sticky Note2               | Sticky Note                    | Note about Leonardo.AI usage       |                               |                               | ## Using Leonardo.AI to generate image Replace `HTTP- Gemini` node with this node. Create new header auth credential with your leonardo.ai API key. |
| Sticky Note3               | Sticky Note                    | General notes on image generation  |                               |                               | You can try using different models for image generation here. Gemini - [Good quality, 90 free trial with GCP account] (With nano-banana) Leonardo.AI - [Better quality, Paid API] |
| Sticky Note4               | Sticky Note                    | Full setup and usage instructions  |                               |                               | # 🎨 Anime Image Generator — Setup & Notes (detailed instructions and tips) |
| Sticky Note5               | Sticky Note                    | Explains purpose of Wait node      |                               |                               | ## Not necessary Used to avoid rate limits                   |
| Sticky Note6               | Sticky Note                    | Explains purpose of Image-count    |                               |                               | ## Generates 4 images for an image query                      |
| Sticky Note7               | Sticky Note                    | Configure chat restriction on Telegram Trigger |                             |                               | ## Configure Me! Paste your own chatID in the "Restrict to Chat IDs" field |
| Sticky Note8               | Sticky Note                    | Configure chat ID for sending pics |                               |                               | ## Configure Me! Paste your own chatID in the "ChatID" field |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node**  
   - Type: Telegram Trigger  
   - Set `updates` to listen to "message".  
   - Leave `chatIds` empty or add your Telegram chat ID(s) to restrict input.  
   - Add your Telegram Bot API credentials (token from BotFather).  

2. **Create Set node "Image-count"**  
   - Extract user prompt text: Set variable `User_Prompt` = `{{$json["message"]["text"]}}`  
   - Set number of images: Set variable `num_of_Images` = 4 (default).  

3. **Create LangChain Chat LLM node "DeepSeek from OpenRouter"**  
   - Model: `deepseek/deepseek-chat-v3-0324:free`  
   - Set temperature to 0.7 and topP to 0.8.  
   - Add OpenRouter API credentials.  

4. **Create LangChain Chain LLM node "Prompt generator (USER_Inspired)"**  
   - Input text: Use template including:  
     - “Enhance My Anime Image Prompt!” instruction with embedded user prompt and number of images.  
     - System message defining expert anime artist role and detailed prompt structure.  
   - Enable output parser.  

5. **Create LangChain Structured Output Parser node**  
   - Schema: JSON array of objects with required string property `description`.  
   - Connect output of "Prompt generator (USER_Inspired)" to this node.  

6. **Create Split Out node "Split Out Prompts"**  
   - Field to split out: `output` (the array of prompt objects).  

7. **Create SplitInBatches node "Loop Over Prompts"**  
   - Default batch size 1 (process each prompt independently).  

8. **Create HTTP Request node "HTTP- Gemini"**  
   - Method: POST  
   - URL: `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash-preview-image-generation:generateContent`  
   - Body type: JSON with structure containing prompt text from `{{$json["description"]}}`.  
   - Request headers: Use Google Palm API credential for authentication.  
   - Request body includes response modalities TEXT and IMAGE.  

9. **Create Set node "Set base64"**  
   - Extract base64 image data from Gemini response JSON nested structure:  
     `{{$json["candidates"][0]["content"]["parts"][1]?.inlineData.data || $json["candidates"][0]["content"]["parts"][0]?.inlineData.data}}`  

10. **Create ConvertToFile node "Convert to File"**  
    - Operation: toBinary  
    - Source property: `base64` (from previous Set node).  

11. **Create Wait node "Wait 10s"**  
    - Amount: 10 seconds (to avoid rate limits).  

12. **Create Telegram node "Send pics via Telegram"**  
    - Operation: sendPhoto  
    - Chat ID: Set your Telegram chat ID here or pass dynamically from trigger.  
    - Use binary data from previous node to send image file.  
    - Use same Telegram Bot API credentials as trigger.  

13. **Connect nodes:**  
    - Telegram Trigger → Image-count → DeepSeek from OpenRouter → Prompt generator (USER_Inspired) → Structured Output Parser → Split Out Prompts → Loop Over Prompts → HTTP- Gemini → Set base64 → Convert to File → Wait 10s → Loop Over Prompts (for next iteration)  
    - Loop Over Prompts → Send pics via Telegram (to send each image)  

14. **Optional: Replace "HTTP- Gemini" node with "HTTP- Leonardo AI" node**  
    - POST to Leonardo.AI API endpoint with required body parameters for anime image generation.  
    - Set HTTP Header Auth with Leonardo.AI API key credential.  
    - Adjust prompt and style parameters per Leonardo.AI documentation.  

15. **Add Sticky Notes**  
    - Create sticky notes with instructions, configuration tips, and usage notes as per original workflow.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                     | Context or Link                                                                                  |
|---------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Create Telegram bot with BotFather and add Bot Token and your Telegram User ID to workflow for chat restrictions.               | Official Telegram BotFather: https://t.me/BotFather                                            |
| Gemini offers free image generation for 90 days with GCP account.                                                               | Requires Google Cloud Platform account and API key.                                            |
| Leonardo.AI provides better quality anime images but requires paid API key and subscription.                                    | API docs: https://cloud.leonardo.ai/api/rest/v1/generations                                    |
| Use short, simple prompts in Telegram to get best results; LLM expands them into detailed anime-style descriptions.             | Workflow tip                                                                                   |
| Avoid NSFW or copyrighted character prompts to comply with provider policies.                                                   | Usage policy note                                                                              |
| Keep API keys private and do not share them in screenshots or public forums.                                                     | Security best practice                                                                         |
| To prevent rate limits, the workflow inserts a 10-second wait between sending images.                                            | See "Wait 10s" node note                                                                      |
| Workflow uses DeepSeek model via OpenRouter for prompt enhancement but can be swapped with other LLM providers.                 | OpenRouter API: https://openrouter.ai                                                          |
| Gemini and Leonardo.AI nodes are interchangeable; replace the Gemini node with Leonardo.AI node and update credentials accordingly. | See sticky notes for instructions                                                             |

---

**Disclaimer:** The provided description and analysis are based on an automated n8n workflow for legal and public data processing. All API usage complies with respective provider policies.
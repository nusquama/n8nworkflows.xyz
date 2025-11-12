Auto Icon & Cover for New Notion Pages

https://n8nworkflows.xyz/workflows/auto-icon---cover-for-new-notion-pages-5141


# Auto Icon & Cover for New Notion Pages

### 1. Workflow Overview

This workflow automates the process of assigning an icon (emoji) and a cover image to newly created Notion pages. It is designed to be triggered by an HTTP webhook, typically called by Notion's automation or a button integration. The workflow fetches a list of images, selects random ones, and uses AI to analyze the page title to suggest an appropriate emoji icon. If the AI fails, it falls back to a default random emoji. Finally, it updates the Notion page with the chosen emoji and cover image.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives incoming webhook requests that notify about new Notion pages.
- **1.2 Random Number Assignment:** Generates random numbers to be used for image selection.
- **1.3 Image Retrieval:** Fetches a list of images from an external API.
- **1.4 Emoji Analysis via AI:** Analyzes the Notion page title with a language model to suggest a suitable emoji.
- **1.5 Default Emoji Selection:** If AI fails or returns no emoji, selects a random emoji as fallback.
- **1.6 Assign Cover Images:** Randomly assigns a cover image URL from the fetched images.
- **1.7 Notion Page Update:** Updates the Notion page with the selected emoji icon and cover image.
- **1.8 Workflow Control:** Manages looping and batch processing of multiple items.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Receive POST requests via webhook containing Notion page creation data.
- **Nodes Involved:** `Webhook`, `Sticky Note`
- **Node Details:**

  - **Webhook**
    - Type: HTTP Webhook
    - Role: Entry point; waits for POST requests from Notion or external trigger.
    - Configuration: Path set to a unique ID; method POST.
    - Input: HTTP request from Notion automation.
    - Output: JSON payload with new page data.
    - Edge cases: Incorrect payload format, unauthorized access, webhook misconfiguration.
    - Sticky Note: Explains usage of webhook for triggering via Notion button or automation.

  - **Sticky Note**
    - Role: Documentation
    - Content: "use webhook so you can send req by Notion Button or Automation"

#### 2.2 Random Number Assignment

- **Overview:** Generates random numbers for image page selection and other randomization needs.
- **Nodes Involved:** `Random Number`
- **Node Details:**

  - **Random Number**
    - Type: Code (JavaScript)
    - Role: Adds a random integer between 1 and 10 as `num` field to each input item.
    - Configuration: Uses helper function `getRandomInt(min, max)` to generate integers.
    - Input: Items from `Webhook`.
    - Output: Items enriched with `num` and copied `id` fields.
    - Edge cases: If input is empty, output will be empty; random generation errors unlikely but possible if code is modified.

#### 2.3 Image Retrieval

- **Overview:** Fetches a paginated list of images from Picsum Photos API for cover image selection.
- **Nodes Involved:** `HTTP Request`
- **Node Details:**

  - **HTTP Request**
    - Type: HTTP Request
    - Role: Calls "https://picsum.photos/v2/list?page={{$json.num}}&limit=100" to get 100 images.
    - Configuration: URL parameterized by `num` field from previous node.
    - Input: Items from `Random Number` node.
    - Output: Array of image metadata including download URLs.
    - Edge cases: API downtime, rate limiting, invalid page numbers, network errors.

#### 2.4 Emoji Analysis via AI

- **Overview:** Uses an AI language model to analyze the Notion page title and suggest a fitting emoji.
- **Nodes Involved:** `Get Notion Title`, `Basic LLM Chain`, `OpenRouter Chat Model`
- **Node Details:**

  - **Get Notion Title**
    - Type: Code
    - Role: Extracts page title text from incoming webhook payload.
    - Configuration: Checks if `properties.Name.title` exists; otherwise fallback to `data.Name`.
    - Input: Items from `emoji` node.
    - Output: Items with extracted `name` field (page title).
    - Edge cases: Missing or malformed title data.

  - **Basic LLM Chain**
    - Type: Langchain LLM Chain node
    - Role: Sends prompt to AI model requesting a single emoji based on page title.
    - Configuration: Prompt includes a fixed emoji list and page title.
    - Input: Items from `Get Notion Title`.
    - Output: Emoji suggestion in JSON format.
    - Edge cases: AI timeout, empty or invalid response, API key issues.

  - **OpenRouter Chat Model**
    - Type: Language Model (OpenRouter)
    - Role: The underlying AI model used by `Basic LLM Chain`.
    - Configuration: Model "deepseek/deepseek-chat-v3-0324:free", JSON response expected.
    - Credentials: Requires OpenRouter API credentials.
    - Edge cases: API quota limits, network errors, model downtime.

#### 2.5 Default Emoji Selection

- **Overview:** Ensures an emoji is assigned by randomly selecting one if AI fails or returns empty.
- **Nodes Involved:** `Choose Default Emoji`, `emoji`, `Sticky Note1`
- **Node Details:**

  - **Choose Default Emoji**
    - Type: Code
    - Role: Checks if AI-selected emoji is undefined and picks a random emoji from the predefined list.
    - Input: Items from `Basic LLM Chain`.
    - Output: Items with guaranteed `emoji` field.
    - Edge cases: Empty input, random selection errors.
    - Sticky Note: "## Default Emoji\nif AI choose is fail, it's will use random emoji"

  - **emoji**
    - Type: Code
    - Role: Contains a full emoji string for use in random selection; assigns emoji to input items.
    - Input: Items from `HTTP Request`.
    - Output: Items enriched with `emoji` field.
    - Edge cases: Emoji list must be valid; no empty emoji allowed.

  - **Sticky Note1**
    - Role: Documentation
    - Content: "## Default Emoji\nif AI choose is fail,it's will use random emoji"

#### 2.6 Assign Cover Images

- **Overview:** Assigns a random cover image URL from the Picsum photos retrieved.
- **Nodes Involved:** `picurl`, `Loop Over Items`
- **Node Details:**

  - **picurl**
    - Type: Code
    - Role: For each item, assigns a random image URL from the HTTP Request node’s output as `picurl`.
    - Input: Items from `Random Number` (with access to `HTTP Request` items).
    - Output: Items enriched with `picurl` field containing image URL.
    - Edge cases: Index out of range if random number exceeds image list size.

  - **Loop Over Items**
    - Type: SplitInBatches
    - Role: Processes items in batches equal to the number of emoji items.
    - Input: Items from `picurl`.
    - Output: Batches to `Update Notion Page` node.
    - Edge cases: Batch size mismatch, empty input.

#### 2.7 Notion Page Update

- **Overview:** Updates the Notion page with the chosen emoji icon and cover image URL.
- **Nodes Involved:** `Update Notion Page`
- **Node Details:**

  - **Update Notion Page**
    - Type: HTTP Request
    - Role: Sends PATCH request to Notion API to update page icon and cover.
    - Configuration:
      - URL: PATCH "https://api.notion.com/v1/pages/{{ page_id }}"
      - Body: JSON containing icon emoji and external cover URL.
      - Auth: Uses Notion API OAuth2 credential.
    - Input: Items from batch splitter `Loop Over Items`.
    - Output: Response from Notion API.
    - Edge cases: Invalid page ID, authentication errors, rate limits, malformed JSON, network failures.

#### 2.8 Workflow Control

- **Overview:** Manages flow and sequencing between nodes.
- **Nodes Involved:** Connections between nodes, no explicit control nodes other than `Loop Over Items`.

---

### 3. Summary Table

| Node Name           | Node Type                          | Functional Role                      | Input Node(s)          | Output Node(s)        | Sticky Note                                                                                      |
|---------------------|----------------------------------|------------------------------------|------------------------|-----------------------|-------------------------------------------------------------------------------------------------|
| Webhook             | HTTP Webhook                     | Entry point, receive Notion data   |                        | Random Number          | use webhook so you can send req by Notion Button or Automation                                  |
| Sticky Note         | Sticky Note                     | Documentation                      |                        |                       | use webhook so you can send req by Notion Button or Automation                                  |
| Random Number       | Code                            | Generate random integers for images| Webhook                 | HTTP Request           |                                                                                                 |
| HTTP Request        | HTTP Request                   | Fetch images from Picsum API       | Random Number           | emoji                  |                                                                                                 |
| emoji               | Code                            | Assign emoji field to items        | HTTP Request            | Get Notion Title       |                                                                                                 |
| Get Notion Title    | Code                            | Extract Notion page title          | emoji                   | Basic LLM Chain        |                                                                                                 |
| Basic LLM Chain     | Langchain LLM Chain             | AI emoji suggestion from title    | Get Notion Title        | Choose Default Emoji   |                                                                                                 |
| OpenRouter Chat Model| Langchain Language Model        | Backend AI model for emoji         | Basic LLM Chain (ai)    | Basic LLM Chain        |                                                                                                 |
| Choose Default Emoji| Code                            | Fallback to random emoji if AI fails | Basic LLM Chain        | picurl                 | ## Default Emoji if AI choose is fail,it's will use random emoji                                |
| Sticky Note1        | Sticky Note                     | Documentation                      |                        |                       | ## Default Emoji if AI choose is fail,it's will use random emoji                                |
| picurl              | Code                            | Assign random cover image URL      | Choose Default Emoji    | Loop Over Items        |                                                                                                 |
| Loop Over Items     | SplitInBatches                  | Batch processing for updates       | picurl                  | Update Notion Page     |                                                                                                 |
| Update Notion Page  | HTTP Request                   | Update Notion page icon and cover | Loop Over Items         | Loop Over Items        |                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**
   - Type: HTTP Webhook
   - Path: Unique string (e.g., "0e0c42ac-ad54-4823-b1d2-592b3bc33337")
   - HTTP Method: POST
   - Purpose: Receive Notion page creation payload.

2. **Create Code Node "Random Number"**
   - Input: Connect from Webhook
   - Type: Code (JavaScript)
   - Code: Generate random integer `num` between 1 and 10 for each item; copy `id` from input.
   - Purpose: Prepare random page number for image API.

3. **Create HTTP Request Node**
   - Input: Connect from Random Number
   - Type: HTTP Request
   - Method: GET
   - URL: `https://picsum.photos/v2/list?page={{$json.num}}&limit=100`
   - Purpose: Fetch 100 images from Picsum Photos API.

4. **Create Code Node "emoji"**
   - Input: Connect from HTTP Request
   - Type: Code
   - Code: Define a string of allowed emojis; assign one emoji per item (random selection).
   - Purpose: Prepare emoji list for AI or fallback.

5. **Create Code Node "Get Notion Title"**
   - Input: Connect from "emoji"
   - Type: Code
   - Code: Extract Notion page title from webhook payload under `body.data.properties.Name.title[0].plain_text` or fallback.
   - Purpose: Extract input text for AI emoji analysis.

6. **Create Basic LLM Chain Node**
   - Input: Connect from "Get Notion Title"
   - Type: Langchain LLM Chain
   - Text Prompt: Request one emoji from a fixed emoji list based on page title.
   - Retry on Fail: Enabled
   - Purpose: Use AI to select an emoji relevant to the page title.

7. **Create OpenRouter Chat Model Node**
   - Credential: Provide OpenRouter API credentials.
   - Model: `deepseek/deepseek-chat-v3-0324:free`
   - Response Format: JSON object
   - Connect as AI model to Basic LLM Chain node.

8. **Create Code Node "Choose Default Emoji"**
   - Input: Connect from Basic LLM Chain
   - Type: Code
   - Code: If AI emoji is undefined or empty, pick a random emoji from the predefined list.
   - Purpose: Provide fallback emoji.

9. **Create Code Node "picurl"**
   - Input: Connect from "Choose Default Emoji"
   - Type: Code
   - Code: For each item, assign a random cover image URL from the `HTTP Request` node’s output.
   - Purpose: Assign cover images randomly.

10. **Create SplitInBatches Node "Loop Over Items"**
    - Input: Connect from "picurl"
    - Batch Size: Set dynamically to the length of emoji list (e.g., `={{ $('emoji').all().length }}`)
    - Purpose: Handle batch processing of updates.

11. **Create HTTP Request Node "Update Notion Page"**
    - Input: Connect from "Loop Over Items"
    - Method: PATCH
    - URL: `https://api.notion.com/v1/pages/{{ $('Webhook').item.json.body.data.id }}`
    - Body Type: JSON
    - Body Content:
      ```json
      {
        "icon": {
          "type": "emoji",
          "emoji": "{{ $('Choose Default Emoji').item.json.emoji }}"
        },
        "cover": {
          "type": "external",
          "external": {
            "url": "{{ $json.picurl }}"
          }
        }
      }
      ```
    - Authentication: Use Notion OAuth2 credentials.
    - Purpose: Update Notion page icon and cover image.

12. **Add Sticky Notes for Documentation**
    - Near Webhook: "use webhook so you can send req by Notion Button or Automation"
    - Near Choose Default Emoji: "## Default Emoji\nif AI choose is fail, it's will use random emoji"

13. **Connect nodes as per the flow:**
    - Webhook → Random Number → HTTP Request → emoji → Get Notion Title → Basic LLM Chain → Choose Default Emoji → picurl → Loop Over Items → Update Notion Page → Loop Over Items (loop back)

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                         |
|------------------------------------------------------------------------------------------------------|---------------------------------------------------------|
| The workflow expects Notion webhook payloads with page creation data in a specific JSON structure.  | Ensure Notion sends correct webhook data.               |
| Picsum Photos API provides free placeholder images; consider API limits and availability.             | https://picsum.photos/                                   |
| AI emoji suggestion uses OpenRouter API; requires valid credentials and quota management.             | https://openrouter.ai/                                   |
| Notion API requires OAuth2 credentials with permissions to update pages.                              | https://developers.notion.com/reference/patch-page      |
| Emojis used must comply with Notion allowed emojis; fallback ensures no empty icon.                   | Workflow includes comprehensive emoji list.             |

---

This documentation covers the entire workflow named *Auto Icon & Cover for New Notion Pages*, enabling detailed understanding, troubleshooting, and full reproduction in n8n without the original JSON.
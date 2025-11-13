Amazon Affiliate Links to Mastodon with AI Descriptions & Shlink Tracking

https://n8nworkflows.xyz/workflows/amazon-affiliate-links-to-mastodon-with-ai-descriptions---shlink-tracking-4935


# Amazon Affiliate Links to Mastodon with AI Descriptions & Shlink Tracking

### 1. Workflow Overview

This workflow automates the process of posting Amazon affiliate links to Mastodon with AI-generated descriptions and tracks the links using Shlink. It is designed for users managing affiliate marketing campaigns who want to enhance posts with AI-generated ad copy, upload product images, and monitor link performance.

**Logical Blocks:**

- **1.1 Input Reception and Initialization:** Trigger and parameter setup for Mastodon API and Google Sheets access.
- **1.2 Data Retrieval from Google Sheets:** Fetch Amazon affiliate links that have not yet been sent.
- **1.3 Image Download and Upload:** Retrieve the product image and upload it to Mastodon to get a media ID.
- **1.4 AI Description Generation:** Generate a two-line technical ad copy for the product link using a language model.
- **1.5 Mastodon Posting:** Post the combined AI description and shortened tracking link to Mastodon with the uploaded image.
- **1.6 Google Sheets Update:** Mark the item as sent and update the sheet with the Shlink tracking URL.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Initialization

- **Overview:**  
  This block initiates the workflow manually and sets critical parameters such as Mastodon API endpoint and authorization token, then triggers data retrieval.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’  
  - Set Parameter  
  - Google Sheets

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - *Type:* Manual Trigger  
    - *Role:* Starts the workflow on manual execution.  
    - *Configuration:* No parameters; simply triggers downstream nodes.  
    - *Connections:* Output → Set Parameter  
    - *Failure modes:* None typical; if disabled, cannot start manually.

  - **Set Parameter**  
    - *Type:* Set Node  
    - *Role:* Defines Mastodon API URL and token credentials as workflow parameters.  
    - *Configuration:*  
      - `mostodon_instance` set to `https://mastodon.social/api/v2/media` (likely a typo in variable name “mostodon”)  
      - `mastodon_token` set to a placeholder token string (masked here).  
    - *Connections:* Output → Google Sheets  
    - *Edge cases:* Invalid or expired Mastodon token will cause posting failures later.

  - **Google Sheets**  
    - *Type:* Google Sheets (Read)  
    - *Role:* Retrieves rows from the affiliate links sheet where "Send" column is "NO" (i.e., unsent links).  
    - *Configuration:*  
      - Document ID and sheet name set to a specific Google Sheets document with Amazon affiliate data.  
      - Filter applied: `Send` column must equal "NO".  
      - Returns first matching row only.  
    - *Connections:* Output → Get Picture  
    - *Edge cases:*  
      - Authentication errors due to OAuth expiration.  
      - No matching rows (empty output) will halt downstream processing or cause errors if not handled.

---

#### 2.2 Image Download and Upload

- **Overview:**  
  Downloads the product image from the URL specified in the Google Sheets data, then uploads it to Mastodon to obtain media ID for attaching to the post.

- **Nodes Involved:**  
  - Get Picture  
  - Picture to Mastodon

- **Node Details:**

  - **Get Picture**  
    - *Type:* HTTP Request  
    - *Role:* Downloads the image binary data from the URL specified in the `PicURL` field of the input JSON.  
    - *Configuration:*  
      - URL dynamically set to `{{$json.PicURL}}`.  
      - Default GET method.  
      - No timeout or retry explicitly configured.  
    - *Connections:* Output → Basic LLM Chain and Picture to Mastodon  
    - *Edge cases:*  
      - Invalid or inaccessible image URL results in HTTP failure.  
      - Large images may cause timeout or memory issues.

  - **Picture to Mastodon**  
    - *Type:* HTTP Request  
    - *Role:* Uploads the image binary data to Mastodon’s media endpoint to get a media ID for the status post.  
    - *Configuration:*  
      - URL set from `Set Parameter` node’s `mostodon_instance` (Mastodon media API URL).  
      - Method: POST with multipart/form-data, sending the image as binary under 'file' parameter.  
      - Authorization header uses Bearer token from `mastodon_token`.  
    - *Connections:* Output → Merge (second input)  
    - *Edge cases:*  
      - Authorization failure if token invalid.  
      - API rate limits or network errors.  
      - Incorrect media upload format could cause rejection.

---

#### 2.3 AI Description Generation

- **Overview:**  
  Generates a concise two-line technical ad copy for the Amazon product link using the OpenRouter language model, then combines the output with the image upload results.

- **Nodes Involved:**  
  - OpenRouter Chat Model  
  - Basic LLM Chain  
  - Merge

- **Node Details:**

  - **OpenRouter Chat Model**  
    - *Type:* LangChain Chat LLM (OpenRouter)  
    - *Role:* Provides advanced language model capabilities to generate ad text.  
    - *Configuration:*  
      - Model selected: `perplexity/sonar` (specific OpenRouter model).  
      - No additional options.  
      - Credential: OpenRouter API key.  
    - *Connections:* Output (ai_languageModel) → Basic LLM Chain (ai_languageModel input)  
    - *Edge cases:*  
      - API key invalid or quota exceeded causes failure.  
      - Model latency or errors may delay processing.

  - **Basic LLM Chain**  
    - *Type:* LangChain LLM Chain  
    - *Role:* Consumes the product page URL (Amazon Link) and prompts the AI to create a two-line ad copy without the phrase "Here is an ad copy".  
    - *Configuration:*  
      - Text prompt: `"Create a two-line technical ad copy based on this product page: {{ $json[\"Amazon Link\"] }} suppress \"Here is an ad copy\" in the output!"`  
      - Uses ai_languageModel input from OpenRouter Chat Model.  
    - *Connections:* Output → Merge (first input)  
    - *Edge cases:*  
      - Failure if input Amazon Link is missing or malformed.  
      - Model output might contain unwanted phrases if prompt misunderstood.

  - **Merge**  
    - *Type:* Merge Node  
    - *Role:* Combines AI generated text and Mastodon media upload response into one JSON object for posting.  
    - *Configuration:*  
      - Mode: Combine by position (aligns first item from each input).  
    - *Connections:* Output → Mastodon  
    - *Edge cases:*  
      - Mismatched input lengths may cause incomplete merges or missing data downstream.

---

#### 2.4 Mastodon Posting

- **Overview:**  
  Posts the AI-generated ad copy combined with the Shlink tracking URL and attached image to Mastodon.

- **Nodes Involved:**  
  - Mastodon

- **Node Details:**

  - **Mastodon**  
    - *Type:* Mastodon Node (API)  
    - *Role:* Posts status updates with media attachments on Mastodon.  
    - *Configuration:*  
      - Status text composed of AI text plus Shlink URL: `={{ $json.text }}\n{{ $('Get Picture').item.json.SHLink }}`  
      - Media IDs set from uploaded image ID retrieved from `Picture to Mastodon` output.  
      - Visibility set to public and sensitive flag false.  
      - OAuth2 credentials linked to Mastodon account.  
    - *Connections:* Output → Google Update  
    - *Edge cases:*  
      - OAuth token expiration or revocation causes posting failure.  
      - Invalid media ID or API limits cause errors.  
      - Empty status text or missing link would post incomplete message.

---

#### 2.5 Google Sheets Update

- **Overview:**  
  Updates the original Google Sheets row to mark the affiliate link as sent and logs the Shlink tracking URL.

- **Nodes Involved:**  
  - Google Update

- **Node Details:**

  - **Google Update**  
    - *Type:* Google Sheets (Update)  
    - *Role:* Updates the row identified by Shlink with `Send` status set to "YES" and stores the Shlink URL.  
    - *Configuration:*  
      - Sheet and document same as initial read.  
      - Matching row by `SHLink` column.  
      - Columns updated:  
        - `Send` set to "YES"  
        - `SHLink` updated with Shlink URL from input JSON  
      - Uses explicit mapping rather than automatic.  
    - *Connections:* None (workflow end).  
    - *Edge cases:*  
      - Update failure if Shlink key missing or row not found.  
      - OAuth token expiration or permission issues.

---

### 3. Summary Table

| Node Name                   | Node Type                        | Functional Role                      | Input Node(s)               | Output Node(s)          | Sticky Note                                                                                   |
|-----------------------------|---------------------------------|------------------------------------|-----------------------------|-------------------------|------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                  | Workflow start trigger               | —                           | Set Parameter           |                                                                                                |
| Set Parameter               | Set                             | Mastodon API URL and token setup   | When clicking ‘Execute workflow’ | Google Sheets           | Mastodon Parameters                                                                           |
| Google Sheets              | Google Sheets (Read)             | Fetch unsent affiliate links       | Set Parameter               | Get Picture             |                                                                                                |
| Get Picture                | HTTP Request                    | Download product image              | Google Sheets               | Basic LLM Chain, Picture to Mastodon |                                                                                                |
| Picture to Mastodon        | HTTP Request                    | Upload image to Mastodon            | Get Picture                 | Merge                   |                                                                                                |
| OpenRouter Chat Model      | LangChain Chat LLM              | Provide AI language model           | —                           | Basic LLM Chain (ai_languageModel input) |                                                                                                |
| Basic LLM Chain            | LangChain LLM Chain             | Generate two-line ad copy           | Get Picture, OpenRouter Chat Model | Merge                   |                                                                                                |
| Merge                     | Merge                           | Combine AI text and media upload    | Basic LLM Chain, Picture to Mastodon | Mastodon                |                                                                                                |
| Mastodon                  | Mastodon API Node               | Post status with image and text     | Merge                      | Google Update           |                                                                                                |
| Google Update             | Google Sheets (Update)           | Mark link sent and update Shlink URL | Mastodon                   | —                       |                                                                                                |
| Sticky Note               | Sticky Note                     | Workflow overview note              | —                           | —                       | ## Send Amazon Affiliate to Mastodon\n### Monitor the Links with https://shlink.io/           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Type: Manual Trigger  
   - No special parameters.

2. **Create Set Node (Set Parameter):**  
   - Define two string parameters:  
     - `mostodon_instance` = `https://mastodon.social/api/v2/media`  
     - `mastodon_token` = Your Mastodon OAuth2 Bearer token (replace placeholder)  
   - Connect Manual Trigger → Set Parameter.

3. **Create Google Sheets Node (Read):**  
   - Operation: Read Rows  
   - Document ID: Your Google Sheets document holding Amazon affiliate data.  
   - Sheet Name: Sheet1 (or appropriate sheet)  
   - Filters: Filter rows where `Send` column equals "NO".  
   - Option: Return first match only.  
   - Credentials: Google Sheets OAuth2 with read permission.  
   - Connect Set Parameter → Google Sheets.

4. **Create HTTP Request Node (Get Picture):**  
   - Method: GET  
   - URL: Dynamic expression: `{{$json.PicURL}}` (from Google Sheets data)  
   - Connect Google Sheets → Get Picture.

5. **Create LangChain Chat LLM Node (OpenRouter Chat Model):**  
   - Model: `perplexity/sonar` (OpenRouter)  
   - Credentials: OpenRouter API key.  
   - No parameters adjusted.  
   - No direct input connection necessary, but link output to Basic LLM Chain node’s ai_languageModel input.

6. **Create LangChain LLM Chain Node (Basic LLM Chain):**  
   - Text prompt:  
     `"Create a two-line technical ad copy based on this product page: {{ $json[\"Amazon Link\"] }} suppress \"Here is an ad copy\" in the output!"`  
   - Use OpenRouter Chat Model node as ai_languageModel input.  
   - Connect Get Picture → Basic LLM Chain (main input).  
   - Connect OpenRouter Chat Model → Basic LLM Chain (ai_languageModel input).

7. **Create HTTP Request Node (Picture to Mastodon):**  
   - Method: POST  
   - URL: Set from `mostodon_instance` parameter (`https://mastodon.social/api/v2/media`)  
   - Content-Type: multipart/form-data  
   - Body Parameters: One parameter named "file", type formBinaryData, input field name "data" (binary data from Get Picture).  
   - Header: Authorization Bearer token from `mastodon_token` parameter.  
   - Connect Get Picture → Picture to Mastodon.

8. **Create Merge Node:**  
   - Mode: Combine  
   - Combine by Position  
   - Connect Basic LLM Chain → Merge (input 1)  
   - Connect Picture to Mastodon → Merge (input 2).

9. **Create Mastodon Node:**  
   - Operation: Post status  
   - Status text: `={{ $json.text }}\n{{ $('Get Picture').item.json.SHLink }}` (combine generated ad copy and Shlink URL)  
   - Media IDs: set to media ID from Picture to Mastodon output (`={{ $('Picture to Mastodon').item.json.id }}`)  
   - Visibility: public  
   - Sensitive: false  
   - Credentials: OAuth2 Mastodon account credentials.  
   - Connect Merge → Mastodon.

10. **Create Google Sheets Node (Update):**  
    - Operation: Update Row  
    - Document and Sheet same as read node.  
    - Matching column: `SHLink`  
    - Update columns:  
      - `Send`: "YES"  
      - `SHLink`: `={{ $('Google Sheets').item.json.SHLink }}`  
    - Credentials: Google Sheets OAuth2 with write permission.  
    - Connect Mastodon → Google Update.

11. **Add Sticky Note:**  
    - Content:  
      ```
      ## Send Amazon Affiliate to Mastodon
      ### Monitor the Links with https://shlink.io/
      ```  
    - Place visually to cover the main nodes.

---

### 5. General Notes & Resources

| Note Content                                               | Context or Link                      |
|------------------------------------------------------------|------------------------------------|
| Workflow posts Amazon affiliate links with AI descriptions | Workflow purpose                   |
| Uses https://shlink.io/ for link tracking                  | Link tracking service              |
| Mastodon media upload endpoint is `https://mastodon.social/api/v2/media` | Mastodon API reference            |
| AI model used: OpenRouter with `perplexity/sonar` model    | OpenRouter API documentation       |
| Google Sheets is used both for input data and updating status | Data source and tracking            |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.
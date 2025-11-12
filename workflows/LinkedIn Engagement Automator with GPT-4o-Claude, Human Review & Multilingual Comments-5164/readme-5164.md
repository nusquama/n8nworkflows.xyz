LinkedIn Engagement Automator with GPT-4o/Claude, Human Review & Multilingual Comments

https://n8nworkflows.xyz/workflows/linkedin-engagement-automator-with-gpt-4o-claude--human-review---multilingual-comments-5164


# LinkedIn Engagement Automator with GPT-4o/Claude, Human Review & Multilingual Comments

---

### 1. Workflow Overview

This workflow automates LinkedIn engagement by generating multilingual comments, reactions, and mentions on LinkedIn posts, combining advanced AI language models (OpenAI GPT-4o-mini and Anthropic Claude 3.5 Haiku) with human review via Telegram. It targets social media managers, community managers, or professionals aiming to maintain authentic and context-aware engagement on LinkedIn posts with multilingual support.

The workflow is logically divided into these blocks:

- **1.1 Input Reception & Validation:** Captures incoming Telegram messages containing LinkedIn post URLs and validates user identity.
- **1.2 Post Data Extraction:** Extracts the LinkedIn post ID from the URL, then fetches the post content and author details via Unipile API.
- **1.3 Language & Mood Analysis:** Uses AI to detect the primary language and emotional tone of the LinkedIn post.
- **1.4 Reaction Determination:** AI selects an appropriate LinkedIn reaction (like, celebrate, support, love, insightful, funny).
- **1.5 Comment Generation:** AI generates a professional, tailored LinkedIn comment based on post content, mood, and language.
- **1.6 Human Review & Approval:** Sends the generated comment and reaction to a Telegram user for approval or disapproval.
- **1.7 Posting to LinkedIn:** Upon approval, posts the comment with mention and reacts to the LinkedIn post via Unipile API.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Validation

- **Overview:** This block receives Telegram messages as triggers and filters messages to ensure only authorized users can proceed.
- **Nodes Involved:**  
  - URL Trigger  
  - Check Telegram User id

- **Node Details:**

| Node Name            | Details                                                                                                           |
|----------------------|-------------------------------------------------------------------------------------------------------------------|
| **URL Trigger**       | Type: Telegram Trigger node. Listens for incoming Telegram messages.| Configured to capture 'message' updates from Telegram. Requires Telegram API credentials.| Input: Telegram message event; Output: Message JSON.| Possible failures: webhook errors, Telegram API rate limits.|
| **Check Telegram User id**| Type: Filter node. Validates user id to allow only authorized users.| Checks if `$json.message.from.id` equals a predefined Telegram user id (e.g., 123456789).| Input: Telegram message from trigger; Output: passes authorized users only.| Failure if user id mismatched or missing.|

#### 2.2 Post Data Extraction

- **Overview:** Extracts the LinkedIn post ID from the provided URL and obtains detailed post content and author info from Unipile API.
- **Nodes Involved:**  
  - Defining guardrails  
  - OpenAI Chat Model  
  - Structured Output Parser  
  - Extract the ID of the LinkedIn post  
  - Extract the content of the LinkedIn post

- **Node Details:**

| Node Name                  | Details                                                                                                                                |
|----------------------------|----------------------------------------------------------------------------------------------------------------------------------------|
| **Defining guardrails**    | Type: Set node. Initializes key variables such as LinkedIn URL, Unipile account ID, API key, and DSN.| Inputs default values; outputs variables for downstream use.| Potential failure: missing or incorrect credentials.|
| **OpenAI Chat Model**      | Type: Langchain OpenAI Chat Model (gpt-4o-mini). Used to process URL text and extract post ID.| Model: gpt-4o-mini, no special options.| Input: LinkedIn post URL; Output parsed by Structured Output Parser.| API key required; potential rate limit or auth failures.|
| **Structured Output Parser**| Type: Langchain Structured Output Parser. Parses JSON output for post ID.| Configured to expect JSON with key "post-id".| Input: raw model output; Output: structured post ID.| Edge case: parsing errors if output malformed.|
| **Extract the ID of the LinkedIn post**| Type: Langchain Agent node. Uses system prompt instructing to locate post id after hyphens in URL.| Input: URL string from variables; Output: post ID.| Requires output parser.| Failure modes: ambiguous URL, incorrect parsing.|
| **Extract the content of the LinkedIn post**| Type: HTTP Request node. Calls Unipile API to fetch post details.| Configured with URL using DSN and extracted post id, includes account ID and API key headers.| Input: post ID; Output: JSON with post text, author name/id, social_id.| API failures, 404 (post not found), or authentication errors.|

#### 2.3 Language & Mood Analysis

- **Overview:** Determines the primary language and emotional tone of the LinkedIn post using AI.
- **Nodes Involved:**  
  - Determine language and mood  
  - OpenAI Chat Model2

- **Node Details:**

| Node Name                  | Details                                                                                                                                |
|----------------------------|----------------------------------------------------------------------------------------------------------------------------------------|
| **Determine language and mood**| Type: Langchain Information Extractor node.| Uses a system prompt specifying strict rules for language and mood detection.| Input: post text; Output attributes: language (lowercase), mood (e.g., neutral, enthusiastic).| Edge cases: posts with emojis only (returns unknown), mixed languages, unclear tone.|
| **OpenAI Chat Model2**     | Type: Langchain OpenAI Chat Model (gpt-4o-mini). Called to process content for language and mood.| Model topP set to 0.1 for focused output.| Credential: OpenAI API.| Potential API or parsing failures.|

#### 2.4 Reaction Determination

- **Overview:** AI selects the best reaction emoji type to apply to the LinkedIn post based on content and tone.
- **Nodes Involved:**  
  - Set reaction  
  - OpenAI Chat Model1  
  - Structured Output Parser2

- **Node Details:**

| Node Name                  | Details                                                                                                                                |
|----------------------------|----------------------------------------------------------------------------------------------------------------------------------------|
| **Set reaction**           | Type: Langchain Agent node with system message defining role as LinkedIn assistant choosing reaction.| Input: full post text.| Output: selected reaction type (one of like, celebrate, support, love, insightful, funny).| Parsed by Structured Output Parser2.|
| **OpenAI Chat Model1**     | Type: Langchain OpenAI Chat Model (gpt-4o-mini). Uses topP 0.1 to narrow choices.| Credential: OpenAI API.| Possible failure: ambiguous reaction choice.|
| **Structured Output Parser2**| Parses JSON output containing the reaction.| Potential failure: malformed JSON.|

#### 2.5 Comment Generation

- **Overview:** Generates a professional, personalized, and multilingual LinkedIn comment that engages the post author.
- **Nodes Involved:**  
  - Set comment and prompt properties  
  - Create comment  
  - Anthropic Chat Model  
  - Structured Output Parser1  
  - Set comment content

- **Node Details:**

| Node Name                      | Details                                                                                                                                |
|--------------------------------|----------------------------------------------------------------------------------------------------------------------------------------|
| **Set comment and prompt properties**| Type: Set node. Aggregates variables including post creator name, post content, social_id, author id, detected mood & language, role description, comment length, and example openers.| Inputs from previous nodes; outputs structured data for comment creation.| Failure: missing or incorrect input variables.|
| **Create comment**             | Type: Langchain Agent node. Uses detailed system message with instructions to generate a comment following strict format and content rules.| Input includes post content and all properties set above.| Output: JSON with "comment" string.| Failure: output not following format or missing "NAME" placeholder.|
| **Anthropic Chat Model**       | Type: Langchain Anthropic Claude 3.5 Haiku model.| Alternative AI model used for comment generation.| Credentials required.| Edge cases: model may not comply with strict formatting; fallback or retry may be needed.|
| **Structured Output Parser1** | Parses AI output to extract structured comment text.| Failure: parsing errors if output format deviates.|
| **Set comment content**        | Type: Set node. Replaces placeholder "NAME" with n8n expression `{0}` for mention handling.| Input: parsed comment string; Output: prepared comment text.| Failure: missing or malformed comment.|

#### 2.6 Human Review & Approval

- **Overview:** Sends generated comment and reaction to Telegram for manual approval before posting.
- **Nodes Involved:**  
  - Approve oder Disapprove  
  - If

- **Node Details:**

| Node Name                      | Details                                                                                                                                |
|--------------------------------|----------------------------------------------------------------------------------------------------------------------------------------|
| **Approve oder Disapprove**   | Type: Telegram node with "sendAndWait" operation.| Sends comment and reaction text to Telegram chat for approval.| Waits for "double" approval (approve or disapprove).| Input: comment from Set comment content.| Possible failure: Telegram API errors or timeout if no response.|
| **If**                       | Type: If node. Checks if JSON path `data.approved` is true.| Routes workflow: if approved, proceeds to posting; else halts.| Failure: missing approval data.|

#### 2.7 Posting to LinkedIn

- **Overview:** Posts the approved comment with mention and applies the selected reaction to the LinkedIn post via Unipile API.
- **Nodes Involved:**  
  - Send comment  
  - Send reaction

- **Node Details:**

| Node Name                 | Details                                                                                                                                |
|---------------------------|----------------------------------------------------------------------------------------------------------------------------------------|
| **Send comment**           | Type: HTTP Request node. Posts comment to Unipile API endpoint `/posts/{social_id}/comments`.| Uses account ID, API key, and DSN from guardrails.| Sends text with mention of post author by name and profile ID.| Failure: API errors, mention formatting errors.|
| **Send reaction**          | Type: HTTP Request node. Posts reaction to Unipile API endpoint `/posts/reaction`.| Uses account ID, post ID, API key.| Reaction type taken from AI output.| Possible failures: invalid reaction type, API error.|

---

### 3. Summary Table

| Node Name                  | Node Type                                   | Functional Role                         | Input Node(s)                      | Output Node(s)                      | Sticky Note                                                                                  |
|----------------------------|---------------------------------------------|---------------------------------------|----------------------------------|-----------------------------------|----------------------------------------------------------------------------------------------|
| URL Trigger                | Telegram Trigger                            | Receive Telegram message               | -                                | Check Telegram User id            |                                                                                              |
| Check Telegram User id     | Filter                                      | Validate authorized Telegram user     | URL Trigger                     | Defining guardrails               |                                                                                              |
| Defining guardrails       | Set                                         | Initialize variables and credentials  | Check Telegram User id           | Extract the ID of the LinkedIn post|                                                                                              |
| OpenAI Chat Model         | Langchain OpenAI Chat Model                  | Extract LinkedIn post ID               | Defining guardrails              | Structured Output Parser          |                                                                                              |
| Structured Output Parser  | Langchain Output Parser                       | Parse post ID from AI output           | OpenAI Chat Model               | Extract the ID of the LinkedIn post|                                                                                              |
| Extract the ID of the LinkedIn post | Langchain Agent                      | Extract post ID from URL               | Structured Output Parser         | Extract the content of the LinkedIn post|                                                                                              |
| Extract the content of the LinkedIn post | HTTP Request                      | Fetch post content & author info       | Extract the ID of the LinkedIn post | Determine language and mood        |                                                                                              |
| Determine language and mood| Langchain Information Extractor              | Detect language and emotional tone    | Extract the content of the LinkedIn post | Set reaction                    |                                                                                              |
| OpenAI Chat Model2        | Langchain OpenAI Chat Model                  | Assist language and mood detection     | Determine language and mood      | Set reaction                     |                                                                                              |
| Set reaction              | Langchain Agent                              | Select LinkedIn reaction type          | Determine language and mood      | Structured Output Parser2         |                                                                                              |
| OpenAI Chat Model1        | Langchain OpenAI Chat Model                  | Reaction determination                 | Set reaction                    | Structured Output Parser2         |                                                                                              |
| Structured Output Parser2 | Langchain Output Parser                       | Parse reaction JSON                    | OpenAI Chat Model1              | Set comment and prompt properties |                                                                                              |
| Set comment and prompt properties | Set                                   | Prepare variables for comment creation| Structured Output Parser2       | Create comment                   |                                                                                              |
| Create comment            | Langchain Agent / Anthropic Chat Model       | Generate LinkedIn comment              | Set comment and prompt properties| Structured Output Parser1         |                                                                                              |
| Structured Output Parser1 | Langchain Output Parser                       | Parse comment JSON                     | Create comment                 | Set comment content              |                                                                                              |
| Set comment content       | Set                                         | Prepare comment text with mention placeholder | Structured Output Parser1      | Approve oder Disapprove          |                                                                                              |
| Approve oder Disapprove   | Telegram (sendAndWait)                        | Human approval of comment & reaction  | Set comment content             | If                              |                                                                                              |
| If                        | If                                          | Conditional route on approval status  | Approve oder Disapprove         | Send comment / Set comment and prompt properties|                                                                                              |
| Send comment              | HTTP Request                                 | Post comment with mention to LinkedIn | If (approved)                  | Send reaction                   |                                                                                              |
| Send reaction             | HTTP Request                                 | Post reaction to LinkedIn post         | Send comment                   | -                               |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**
   - Type: Telegram Trigger
   - Set webhook to listen for "message" updates.
   - Configure Telegram API credentials.
   
2. **Add Filter Node: Check Telegram User id**
   - Type: Filter
   - Condition: `$json.message.from.id == YOUR_TELEGRAM_USER_ID`
   - Connect from Telegram Trigger.

3. **Add Set Node: Defining guardrails**
   - Define variables:  
     - `URL_of_the_LinkedIn_post` = `{{$json.message.text}}`  
     - `unipile_account_id` = "YOUR UNIPILE ACCOUNT ID"  
     - `unipile_X-API-KEY` = "YOUR UNIPILE X-API KEY"  
     - `unipile_DSN` = "YOUR UNIPILE DSN"  
   - Connect from Filter node.

4. **Add Langchain OpenAI Chat Model Node**
   - Model: gpt-4o-mini
   - Purpose: extract post ID from URL
   - Input: `{{$json.URL_of_the_LinkedIn_post}}`
   - Connect from Defining guardrails.

5. **Add Langchain Structured Output Parser Node**
   - JSON schema example: `{ "post-id": "123456" }`
   - Connect from OpenAI Chat Model.

6. **Add Langchain Agent Node: Extract the ID of the LinkedIn post**
   - System message: "Find out the post id of the LinkedIn post. Return only the post id. If unsure, look for hyphens. The post id is usually right after it."
   - Input: from Structured Output Parser.
   - Connect from Structured Output Parser.

7. **Add HTTP Request Node: Extract the content of the LinkedIn post**
   - Method: GET
   - URL: `https://{{ unipile_DSN }}/api/v1/posts/{{ $json.output['post-id'] }}`
   - Query Parameter: `account_id = unipile_account_id`
   - Header Parameters: `X-API-KEY = unipile_X-API-KEY`, `accept = application/json`
   - Connect from Extract the ID of the LinkedIn post.

8. **Add Langchain Information Extractor Node: Determine language and mood**
   - System prompt: as per workflow text specifying strict rules for language and mood detection.
   - Attributes: `language`, `mood`
   - Input: post text from HTTP Request node.
   - Connect from Extract the content of the LinkedIn post.

9. **Add Langchain OpenAI Chat Model Node (OpenAI Chat Model2)**
   - Model: gpt-4o-mini (topP 0.1)
   - Input: from Determine language and mood node.
   - Connect from Determine language and mood.

10. **Add Langchain Agent Node: Set reaction**
    - System message defining LinkedIn assistant role selecting reaction from given list.
    - Input: full post text.
    - Connect from OpenAI Chat Model2.

11. **Add Langchain OpenAI Chat Model Node (OpenAI Chat Model1)**
    - Model: gpt-4o-mini (topP 0.1)
    - Connect from Set reaction.

12. **Add Structured Output Parser Node (Structured Output Parser2)**
    - JSON schema example: `{ "reaction": "like" }`
    - Connect from OpenAI Chat Model1.

13. **Add Set Node: Set comment and prompt properties**
    - Assign variables from previous nodes: author name, post content, social_id, author id, detected mood, language, role description, comment length, example openers (these are user-provided configuration strings).
    - Connect from Structured Output Parser2.

14. **Add Langchain Agent Node: Create comment**
    - System message: detailed instructions for comment generation including tone, language, length, structure, openers, and placeholders.
    - Input: from Set comment and prompt properties.
    - Connect from Set comment and prompt properties.

15. **Add Langchain Anthropic Chat Model Node (optional alternative)**
    - Model: Claude 3.5 Haiku
    - Connect from Set comment and prompt properties (alternative to OpenAI model).
   
16. **Add Structured Output Parser Node (Structured Output Parser1)**
    - JSON schema example: `{ "comment": "Your comment text" }`
    - Connect from Create comment node.

17. **Add Set Node: Set comment content**
    - Replace "NAME" placeholder in comment with `{0}` for mention formatting.
    - Connect from Structured Output Parser1.

18. **Add Telegram Node: Approve oder Disapprove**
    - Operation: sendAndWait
    - Chat ID: `{{$json.message.chat.id}}`
    - Message: includes generated comment and reaction.
    - Configure Telegram API credentials.
    - Connect from Set comment content.

19. **Add If Node**
    - Condition: check if `$json.data.approved == true`
    - Connect from Approve oder Disapprove.

20. **Add HTTP Request Node: Send comment**
    - Method: POST
    - URL: `https://{{ unipile_DSN }}/api/v1/posts/{{social_id}}/comments`
    - Body (JSON): includes account_id, text (with replaced newlines), mentions array with author name and profile_id.
    - Headers: X-API-KEY, accept application/json.
    - Connect from If node (true branch).

21. **Add HTTP Request Node: Send reaction**
    - Method: POST
    - URL: `https://{{ unipile_DSN }}/api/v1/posts/reaction`
    - Body Parameters: account_id, post_id, reaction_type (from AI output)
    - Headers: X-API-KEY, accept application/json.
    - Connect from Send comment.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                 | Context or Link                                                                                         |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| The workflow respects strict content policies and only processes legal, public data from LinkedIn posts and Telegram messages.                              | Disclaimer                                                                                             |
| The "NAME" placeholder in comments is replaced with `{0}` to allow dynamic mention formatting supported by Unipile API.                                    | Mention formatting detail                                                                              |
| Use of two AI providers (OpenAI GPT-4o-mini and Anthropic Claude 3.5 Haiku) provides redundancy and model selection flexibility for comment generation.    | AI model strategy                                                                                      |
| Telegram integration includes manual double approval for quality control, avoiding unwanted or inappropriate automated posting.                             | Human review mechanism                                                                                 |
| Useful links for similar workflows and setup: https://n8n.io/workflows/12345 (example placeholder, replace with your actual workflow references if any)     | External resources                                                                                     |

---

**Disclaimer:** The provided text is extracted exclusively from an automated workflow created with n8n, an integration and automation tool. This workflow complies strictly with current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.

---
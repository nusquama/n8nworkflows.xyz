Generate and score LinkedIn posts from bullet points with Gemini and Google Sheets

https://n8nworkflows.xyz/workflows/generate-and-score-linkedin-posts-from-bullet-points-with-gemini-and-google-sheets-20533


# Generate and score LinkedIn posts from bullet points with Gemini and Google Sheets

### 1. Workflow Overview

This workflow is designed to automate the ideation, generation, multi-metric scoring, logging, and presentation of LinkedIn content. By taking unstructured bullet points as input, it leverages Google Gemini to synthesize three distinct post variations, evaluates their marketing potential through rigorous prompt-based scoring, records the metadata and outputs in a centralized Google Sheet, and finally delivers a fully formatted HTML user interface displaying the winning and alternative variations.

The workflow logic is categorized into six functional blocks:
- **1.1 Input Reception & Normalization:** Captures HTTP POST requests via a webhook and standardizes configuration variables (such as tone, post type, author credentials, and target sheet parameters).
- **1.2 AI Post Generation:** Utilizes a Google Gemini language model chain coupled with a structured output parser to produce three distinct LinkedIn post variations constrained by strict character and stylistic rules.
- **1.3 AI Post Scoring:** Passes the generated variations through a secondary Google Gemini language model chain to evaluate hook strength, readability, and engagement potential.
- **1.4 Selection & Data Transformation:** Executes a custom JavaScript routine to calculate cumulative scores, identify the highest-performing post, and sanitize strings for safe HTML rendering.
- **1.5 Data Persistence:** Appends all generated variations, sub-scores, aggregate metrics, and metadata to a designated Google Sheets worksheet.
- **1.6 Response Delivery:** Renders a responsive HTML template containing the highlighted recommendation badge alongside the alternative options and relays it back to the webhook caller.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Normalization
- **Overview:** Acts as the gateway for the automation, capturing raw input payloads and assigning dependable default values for subsequent processing nodes.
- **Nodes Involved:** 
  - `When Bullet Points Submitted`
  - `Set Post Parameters`
- **Node Details:**
  - **When Bullet Points Submitted**
    - *Type and Technical Role:* Webhook Trigger (`n8n-nodes-base.webhook`)
    - *Configuration Choices:* Listens for incoming HTTP `POST` requests on path `linkedin-generator` using manual response handling via a response node.
    - *Key Expressions or Variables:* None (entry point).
    - *Input and Output Connections:* Input: None; Output: `Set Post Parameters`.
    - *Version-Specific Requirements:* TypeVersion 2.
    - *Edge Cases / Potential Failures:* Client timeouts or incorrect HTTP methods (must receive `POST`).
  - **Set Post Parameters**
    - *Type and Technical Role:* Edit Fields / Set Node (`n8n-nodes-base.set`)
    - *Configuration Choices:* Maps incoming webhook properties to explicit variables while establishing fallbacks.
    - *Key Expressions or Variables:* 
      - `bullet_points`: `={{ $json.body.bullet_points }}`
      - `tone`: `={{ $json.body.tone || 'Professional' }}`
      - `post_type`: `={{ $json.body.post_type || 'Thought leadership' }}`
      - `author_name`: `={{ $json.body.author_name || '' }}`
      - `sheet_id`: `REPLACE_WITH_GOOGLE_SHEET_ID`
      - `sheet_name`: `Posts`
    - *Input and Output Connections:* Input: `When Bullet Points Submitted`; Output: `Generate Post Variations`.
    - *Version-Specific Requirements:* TypeVersion 3.4.
    - *Edge Cases / Potential Failures:* Missing body fields handled gracefully by default JavaScript logical OR operators.

#### 2.2 AI Post Generation
- **Overview:** Communicates with Google Gemini to synthesize three distinct LinkedIn post drafts adhering to strict structural, length, and formatting guidelines.
- **Nodes Involved:**
  - `Generate Post Variations`
  - `Gemini Generate Variations`
  - `Parse Variations Output`
- **Node Details:**
  - **Generate Post Variations**
    - *Type and Technical Role:* Basic LLM Chain (`@n8n/n8n-nodes-langchain.chainLlm`)
    - *Configuration Choices:* Defined prompt requesting three distinct post options under 1300 characters, incorporating upstream settings for tone, post type, and author sign-off.
    - *Key Expressions or Variables:* References parameters from `$('Set Post Parameters')`.
    - *Input and Output Connections:* Input: `Set Post Parameters`, `Gemini Generate Variations` (AI Model), `Parse Variations Output` (Output Parser); Output: `Score Variations`.
    - *Version-Specific Requirements:* TypeVersion 1.8.
    - *Edge Cases / Potential Failures:* Rate limiting, API key exhaustion, or non-adherence to the output schema.
  - **Gemini Generate Variations**
    - *Type and Technical Role:* Google Gemini Chat Model (`@n8n/n8n-nodes-langchain.lmChatGoogleGemini`)
    - *Configuration Choices:* Uses standard options coupled with dedicated credentials.
    - *Input and Output Connections:* Output: Connected to `Generate Post Variations` via `ai_languageModel`.
    - *Credentials:* Google Gemini (PaLM) API account.
  - **Parse Variations Output**
    - *Type and Technical Role:* Structured Output Parser (`@n8n/n8n-nodes-langchain.outputParserStructured`)
    - *Configuration Choices:* Manual schema definition expecting a JSON object with properties `variation_1`, `variation_2`, and `variation_3`.
    - *Input and Output Connections:* Output: Connected to `Generate Post Variations` via `ai_outputParser`.

#### 2.3 AI Post Scoring
- **Overview:** Evaluates the generated drafts independently across three key performance metrics: hook strength, readability, and engagement potential.
- **Nodes Involved:**
  - `Score Variations`
  - `Gemini Score Variations`
  - `Parse Scores Output`
- **Node Details:**
  - **Score Variations**
    - *Type and Technical Role:* Basic LLM Chain (`@n8n/n8n-nodes-langchain.chainLlm`)
    - *Configuration Choices:* Prompt instructs the model to score each variation from 1–10 on hook strength, readability, and engagement potential.
    - *Key Expressions or Variables:* Pulls text outputs from `{{ $json.output.variation_1 }}` (and variations 2 & 3).
    - *Input and Output Connections:* Input: `Generate Post Variations`, `Gemini Score Variations`, `Parse Scores Output`; Output: `Select Best Variation`.
    - *Version-Specific Requirements:* TypeVersion 1.8.
  - **Gemini Score Variations**
    - *Type and Technical Role:* Google Gemini Chat Model (`@n8n/n8n-nodes-langchain.lmChatGoogleGemini`)
    - *Configuration Choices:* Standard options configuration.
    - *Input and Output Connections:* Output: Connected to `Score Variations` via `ai_languageModel`.
    - *Credentials:* Google Gemini (PaLM) API account.
  - **Parse Scores Output**
    - *Type and Technical Role:* Structured Output Parser (`@n8n/n8n-nodes-langchain.outputParserStructured`)
    - *Configuration Choices:* Manual schema enforcing nested numeric score objects for `post_1`, `post_2`, and `post_3`.
    - *Input and Output Connections:* Output: Connected to `Score Variations` via `ai_outputParser`.

#### 2.4 Selection & Data Transformation
- **Overview:** Aggregates individual scores, determines the top-performing variation, and structures the payload while escaping HTML entities.
- **Nodes Involved:**
  - `Select Best Variation`
- **Node Details:**
  - **Select Best Variation**
    - *Type and Technical Role:* Code Node (`n8n-nodes-base.code`)
    - *Configuration Choices:* Custom JavaScript to compute total scores, perform conditional evaluation to find the winner index, and sanitize output strings via an HTML-escaping helper function.
    - *Key Expressions or Variables:* Reads upstream node outputs via `$('Generate Post Variations').first().json.output` and `$json.output`.
    - *Input and Output Connections:* Input: `Score Variations`; Output: `Append to Google Sheets`.
    - *Version-Specific Requirements:* TypeVersion 2.
    - *Edge Cases / Potential Failures:* Malformed JSON objects returned by LLM parsers could trigger undefined reference errors.

#### 2.5 Data Persistence
- **Overview:** Records complete execution metrics, content variations, and winning metadata into a centralized Google Sheets tracking document.
- **Nodes Involved:**
  - `Append to Google Sheets`
- **Node Details:**
  - **Append to Google Sheets**
    - *Type and Technical Role:* Google Sheets Integration (`n8n-nodes-base.googleSheets`)
    - *Configuration Choices:* Operation set to `append`, mapping specific column fields to input parameters and evaluation results.
    - *Key Expressions or Variables:* Uses expressions mapping to upstream properties (e.g., `={{ $json.score_1 }}`, `={{ $now.toISO() }}`, `={{ $('Set Post Parameters').item.json.tone }}`).
    - *Input and Output Connections:* Input: `Select Best Variation`; Output: `Respond with Results`.
    - *Version-Specific Requirements:* TypeVersion 4.5.
    - *Credentials:* Google Sheets account (OAuth2).
    - *Edge Cases / Potential Failures:* Invalid Sheet ID, renamed worksheets, or mismatched column header names.

#### 2.6 Response Delivery
- **Overview:** Renders an HTML response containing the recommendation card, score badges, breakdown metrics, and alternative drafts, returning it to the original caller.
- **Nodes Involved:**
  - `Respond with Results`
- **Node Details:**
  - **Respond with Results**
    - *Type and Technical Role:* Respond to Webhook Node (`n8n-nodes-base.respondToWebhook`)
    - *Configuration Choices:* Responds with `text` (HTML format) and injects a custom `Content-Type: text/html` response header.
    - *Key Expressions or Variables:* Injects HTML template strings populated with variables such as `{{ $json.recommended_index }}`, `{{ $json.recommended_score }}`, and variation text blocks.
    - *Input and Output Connections:* Input: `Append to Google Sheets`; Output: None (Terminal node).
    - *Version-Specific Requirements:* TypeVersion 1.1.
    - *Edge Cases / Potential Failures:* Large payloads or malformed HTML templates causing rendering breaks on the client side.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `When Bullet Points Submitted` | `n8n-nodes-base.webhook` | Webhook entry point | None | `Set Post Parameters` | ## Generate LinkedIn posts from bullet points with Gemini<br><br>### How it works<br><br>This workflow receives bullet points through a webhook, applies configurable post settings, and uses Gemini to generate three LinkedIn post variations. It then uses a second Gemini step to score those variations, runs code to select the best one, logs the result to Google Sheets, and returns the generated content to the webhook caller.<br><br>### Setup steps<br><br>- Configure the webhook URL and request payload so it sends the expected bullet points and any optional tone, post type, or author values.<br>- Add Google Gemini credentials for both LLM chain nodes and verify the models are available in your n8n instance.<br>- Connect Google Sheets credentials and select the target spreadsheet, sheet, and columns used for logging the generated and selected post data.<br>- Review the Set Configuration node defaults for bullet_points, tone, post_type, and author_name before activating the workflow.<br><br>### Customization<br><br>Adjust the post-generation prompt, scoring criteria, default tone, post type, and Google Sheets columns to match your LinkedIn content style and reporting needs. |
| `Set Post Parameters` | `n8n-nodes-base.set` | Normalizes incoming payload and sets config defaults | `When Bullet Points Submitted` | `Generate Post Variations` | ## Receive post inputs<br><br>Webhook entry point and nearby configuration step that collect submitted bullet points and normalize the key settings used by the rest of the workflow. |
| `Generate Post Variations` | `@n8n/n8n-nodes-langchain.chainLlm` | LLM Chain generating three LinkedIn drafts | `Set Post Parameters`, `Gemini Generate Variations`, `Parse Variations Output` | `Score Variations` | ## Generate post variations<br><br>Gemini-powered LLM chain that creates three LinkedIn post options from the configured bullet points, with a structured parser to format the generated variations. |
| `Gemini Generate Variations` | `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` | Google Gemini Chat model provider | None | `Generate Post Variations` | ## Generate post variations<br><br>Gemini-powered LLM chain that creates three LinkedIn post options from the configured bullet points, with a structured parser to format the generated variations. |
| `Parse Variations Output` | `@n8n/n8n-nodes-langchain.outputParserStructured` | Enforces JSON schema for the three generated drafts | None | `Generate Post Variations` | ## Generate post variations<br><br>Gemini-powered LLM chain that creates three LinkedIn post options from the configured bullet points, with a structured parser to format the generated variations. |
| `Score Variations` | `@n8n/n8n-nodes-langchain.chainLlm` | LLM Chain scoring the generated drafts | `Generate Post Variations`, `Gemini Score Variations`, `Parse Scores Output` | `Select Best Variation` | ## Score generated variations<br><br>Second Gemini LLM cluster that evaluates all three generated posts and parses the scores into a structured format for comparison. |
| `Gemini Score Variations` | `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` | Google Gemini Chat model provider for scoring | None | `Score Variations` | ## Score generated variations<br><br>Second Gemini LLM cluster that evaluates all three generated posts and parses the scores into a structured format for comparison. |
| `Parse Scores Output` | `@n8n/n8n-nodes-langchain.outputParserStructured` | Enforces numeric score object schema | None | `Score Variations` | ## Score generated variations<br><br>Second Gemini LLM cluster that evaluates all three generated posts and parses the scores into a structured format for comparison. |
| `Select Best Variation` | `n8n-nodes-base.code` | JavaScript code evaluating scores and selecting winner | `Score Variations` | `Append to Google Sheets` | ## Select winning post<br><br>Custom code step that compares the scored variations and chooses the best LinkedIn post to continue through the workflow. |
| `Append to Google Sheets` | `n8n-nodes-base.googleSheets` | Logs workflow runs and variations to Sheets | `Select Best Variation` | `Respond with Results` | ## Log and respond<br><br>Final output cluster that records the selected result in Google Sheets and sends the completed response back to the original webhook request. |
| `Respond with Results` | `n8n-nodes-base.respondToWebhook` | Returns formatted HTML response to client | `Append to Google Sheets` | None | ## Log and respond<br><br>Final output cluster that records the selected result in Google Sheets and sends the completed response back to the original webhook request. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Webhook Trigger Node:**
   - Add a **Webhook** node (`n8n-nodes-base.webhook`).
   - Configure HTTP Method as `POST`, path as `linkedin-generator`, and Response Mode to `Response Node`.
2. **Create the Parameter Configuration Node:**
   - Add a **Set** (`n8n-nodes-base.set`) node connected to the webhook output.
   - Add string assignments:
     - `bullet_points`: `={{ $json.body.bullet_points }}`
     - `tone`: `={{ $json.body.tone || 'Professional' }}`
     - `post_type`: `={{ $json.body.post_type || 'Thought leadership' }}`
     - `author_name`: `={{ $json.body.author_name || '' }}`
     - `sheet_id`: `REPLACE_WITH_GOOGLE_SHEET_ID`
     - `sheet_name`: `Posts`
3. **Set Up Post Generation AI Cluster:**
   - Add a **Basic LLM Chain** node (`@n8n/n8n-nodes-langchain.chainLlm`) named `Generate Post Variations`.
   - Add a **Google Gemini Chat Model** node (`@n8n/n8n-nodes-langchain.lmChatGoogleGemini`) and link it to the LLM chain via the `ai_languageModel` connection. Attach your **Google Gemini (PaLM) API** credentials.
   - Add a **Structured Output Parser** node (`@n8n/n8n-nodes-langchain.outputParserStructured`) and link it via `ai_outputParser`. Configure it with a manual JSON schema requiring properties `variation_1`, `variation_2`, and `variation_3` (all of type string).
   - In the `Generate Post Variations` text parameter, paste the prompt instructing the model to generate three distinct LinkedIn posts under 1300 characters using upstream variables.
4. **Set Up Post Scoring AI Cluster:**
   - Add a second **Basic LLM Chain** node named `Score Variations`.
   - Connect another **Google Gemini Chat Model** node via `ai_languageModel`, reusing your Gemini credentials.
   - Attach a second **Structured Output Parser** node via `ai_outputParser`. Configure its manual schema to expect nested objects for `post_1`, `post_2`, and `post_3`, each containing numeric properties: `hook_strength`, `readability`, and `engagement_potential`.
   - Configure the LLM chain text parameter to ingest the three variations and evaluate them independently.
5. **Create the Selection Code Node:**
   - Add a **Code** node (`n8n-nodes-base.code`) named `Select Best Variation`.
   - Populate the JavaScript code block to calculate score totals, select the highest-scoring variation index, HTML-escape strings, and return the structured JSON payload.
6. **Configure Data Persistence:**
   - Add a **Google Sheets** node (`n8n-nodes-base.googleSheets`) named `Append to Google Sheets`.
   - Connect your **Google Sheets OAuth2 API** credentials.
   - Set the operation to `append`, mapping the document ID and sheet name parameters to expressions reading from `Set Post Parameters`, and mapping columns (`generated_at`, `bullet_points`, `tone`, `post_type`, `variation_1`, `score_1`, etc.) to incoming execution data.
7. **Configure Response Handling:**
   - Add a **Respond to Webhook** node (`n8n-nodes-base.respondToWebhook`) named `Respond with Results`.
   - Set response headers with entry `Content-Type` set to `text/html`.
   - Provide the complete HTML template string in the response body field to render the summary interface.
8. **Finalize Connections:**
   - Ensure node-to-node execution flows sequentially from Webhook $\rightarrow$ Set $\rightarrow$ Generate Chain $\rightarrow$ Score Chain $\rightarrow$ Code $\rightarrow$ Google Sheets $\rightarrow$ Respond to Webhook.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Workflow automation built completely via n8n integration ecosystem. | Process executed with strict adherence to public data usage and platform safety guidelines. |
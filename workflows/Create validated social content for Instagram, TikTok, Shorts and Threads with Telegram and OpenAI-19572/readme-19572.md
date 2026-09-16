Create validated social content for Instagram, TikTok, Shorts and Threads with Telegram and OpenAI

https://n8nworkflows.xyz/workflows/create-validated-social-content-for-instagram--tiktok--shorts-and-threads-with-telegram-and-openai-19572


# Create validated social content for Instagram, TikTok, Shorts and Threads with Telegram and OpenAI

### 1. Workflow Overview

This workflow acts as an automated multi-platform social media content assistant powered by Telegram and OpenAI. Users send a content topic via a Telegram bot, which is validated, enriched with predefined brand and content rules, processed by an AI model, scored for quality, iteratively corrected if flaws are found, and finally formatted into a structured report sent back to the originating Telegram chat.

The logic is grouped into the following functional blocks:
- **1.1 Input Reception & Normalization:** Captures incoming webhooks from Telegram and normalizes the message text and chat ID.
- **1.2 Input Validation & Preferences Enrichment:** Validates the topic length and format (filtering out short strings or bot commands) and assigns global content preferences (tone, audience, language, etc.).
- **1.3 AI Content Generation & Parsing:** Submits the parameters to OpenAI (`gpt-5-mini`) to generate platform-specific drafts, then cleans and parses the JSON response.
- **1.4 Quality Control & Automated Retries:** Evaluates the generated copy against strict platform rules (character limits, hashtag counts, emoji presence, and cross-platform duplication), routes failed outputs through a conditional retry loop (up to two attempts using `gpt-5-mini`), and compiles final success or failure reports.
- **1.5 Telegram Response Dispatch:** Formats the final report into a human-readable message, checks for a valid chat ID, and sends the response back to the user via Telegram.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Reception & Normalization
- **Overview:** Captures incoming messages via a Telegram webhook trigger and maps the raw payload into clean, standardized properties (`topic` and `chat_id`).
- **Nodes Involved:** 
  - `When Topic Received in Telegram`
  - `Set Telegram Input Data`
- **Node Details:**
  - **When Topic Received in Telegram**
    - *Type and Role:* `n8n-nodes-base.telegramTrigger` (Trigger). Listens for incoming message updates from the Telegram Bot API.
    - *Configuration:* Listens to `message` updates.
    - *Input/Output:* Output connects to `Set Telegram Input Data`.
    - *Edge Cases:* Webhook registration failure if bot credentials are invalid or network is unreachable.
  - **Set Telegram Input Data**
    - *Type and Role:* `n8n-nodes-base.set` (Data Transformation). Extracts and renames properties.
    - *Configuration:* Maps `topic` to `={{ $json.message.text }}` and `chat_id` to `={{ $json.message.chat.id }}`. Preserves other incoming fields.
    - *Input/Output:* Input from `When Topic Received in Telegram`; output to `Check Telegram Topic Validity`.
    - *Edge Cases:* Missing `message` object or non-text messages causing undefined evaluations.

#### Block 1.2: Input Validation & Preferences Enrichment
- **Overview:** Ensures the user-provided topic meets minimum length requirements and does not start with a command, routing invalid requests to a help message and valid requests to configuration setup.
- **Nodes Involved:**
  - `Check Telegram Topic Validity`
  - `Send Usage Help via Telegram`
  - `Set Content Preferences`
- **Node Details:**
  - **Check Telegram Topic Validity**
    - *Type and Role:* `n8n-nodes-base.if` (Branching). Evaluates input criteria.
    - *Configuration:* Condition verifies `typeof $json.topic === 'string' && $json.topic.trim().length >= 10 && !$json.topic.trim().startsWith('/')`.
    - *Input/Output:* Input from `Set Telegram Input Data`. True output goes to `Set Content Preferences`; false output goes to `Send Usage Help via Telegram`.
  - **Send Usage Help via Telegram**
    - *Type and Role:* `n8n-nodes-base.telegram` (Action). Sends guidance instructions back to the user.
    - *Configuration:* Sends static help text to `={{ $json.chat_id }}` with `appendAttribution` disabled.
    - *Input/Output:* Input from the false branch of `Check Telegram Topic Validity`.
    - *Edge Cases:* Invalid or expired `chat_id` resulting in a Telegram API error.
  - **Set Content Preferences**
    - *Type and Role:* `n8n-nodes-base.set` (Data Enrichment). Injects default configurations for content generation.
    - *Configuration:* Assigns static values for `tone` (Professional), `target_audience` (General audience), `language` (Same as topic), `call_to_action` (Learn more), `include_emojis` (true), `include_hashtags` (true), and `brand_voice` (Clear, natural and engaging).
    - *Input/Output:* Input from the true branch of `Check Telegram Topic Validity`; output to `OpenAI Generate Captions`.

#### Block 1.3: AI Content Generation & Parsing
- **Overview:** Sends the topic and structured preferences to OpenAI to produce tailored social media copy, then sanitizes and parses the raw model text into JSON.
- **Nodes Involved:**
  - `OpenAI Generate Captions`
  - `Parse Captions Output`
- **Node Details:**
  - **OpenAI Generate Captions**
    - *Type and Role:* `@n8n/n8n-nodes-langchain.openAi` (AI / LLM Integration). Generates text completions based on system instructions and variables.
    - *Configuration:* Uses model `gpt-5-mini`. Prompt injects `{{ $json.topic }}` and preferences (`tone`, `target_audience`, `language`, `brand_voice`, `call_to_action`, `include_emojis`, `include_hashtags`) with explicit platform rules for Instagram, TikTok, YouTube Shorts, and Threads, requiring strict JSON-only output.
    - *Input/Output:* Input from `Set Content Preferences`; output to `Parse Captions Output`.
    - *Edge Cases:* OpenAI API rate limits, timeouts, or failure to output valid JSON.
  - **Parse Captions Output**
    - *Type and Role:* `n8n-nodes-base.code` (Data Transformation / JavaScript). Cleans markdown blocks and parses the model's text payload into structured objects.
    - *Configuration:* Strips ```json wrappers, parses JSON, and merges the result with original metadata from `Set Content Preferences`.
    - *Input/Output:* Input from `OpenAI Generate Captions`; output to `Analyze Content Quality`.
    - *Edge Cases:* Syntax errors if the AI returns malformed JSON, caught via a `try/catch` block throwing an explicit error.

#### Block 1.4: Quality Control & Automated Retries
- **Overview:** Evaluates generated copy against platform-specific constraints (character counts, hashtag counts, emoji rules, and cross-platform duplication), manages regeneration retries if issues are found, and builds success or failure reports.
- **Nodes Involved:**
  - `Analyze Content Quality`
  - `Check Content Quality Status`
  - `Validate Social Content`
  - `Compile Content Report`
  - `Check Retry Limit`
  - `OpenAI Regenerate Captions`
  - `Compile Failed Report`
- **Node Details:**
  - **Analyze Content Quality**
    - *Type and Role:* `n8n-nodes-base.code` (Data Transformation / JavaScript). Computes a quality score (starting at 100) and compiles an array of rule violations.
    - *Configuration:* Checks required fields, character limits (Instagram: 600, TikTok: 180, YT Title: 70, YT Description: 250, Threads: 300), exact hashtag requirements, emoji presence flags, and normalizes text bodies to detect cross-platform duplication. Sets `quality_status` to `passed` or `needs_regeneration`.
    - *Input/Output:* Input from `Parse Captions Output`; output to `Check Content Quality Status`.
  - **Check Content Quality Status**
    - *Type and Role:* `n8n-nodes-base.if` (Branching). Evaluates quality status.
    - *Configuration:* Checks if `{{ $json.quality_status }}` equals `passed`.
    - *Input/Output:* Input from `Analyze Content Quality`. True branch goes to `Validate Social Content`; false branch goes to `Check Retry Limit`.
  - **Validate Social Content**
    - *Type and Role:* `n8n-nodes-base.code` (Data Transformation / JavaScript). Enforces strict final assertions on successful items.
    - *Configuration:* Re-runs validation checks to ensure zero anomalies exist before compiling reports, throwing errors if limits are violated.
    - *Input/Output:* Input from the true branch of `Check Content Quality Status`; output to `Compile Content Report`.
  - **Compile Content Report**
    - *Type and Role:* `n8n-nodes-base.code` (Data Transformation / JavaScript). Structures the validated content pack into a clean, hierarchical payload.
    - *Configuration:* Organizes captions, character lengths, hashtag counts, and quality metrics into a standardized report object.
    - *Input/Output:* Input from `Validate Social Content`; output to `Format Telegram Message`.
  - **Check Retry Limit**
    - *Type and Role:* `n8n-nodes-base.if` (Branching). Limits automatic regeneration loops.
    - *Configuration:* Evaluates `={{ $runIndex < 2 }}` to allow a maximum of two retry iterations.
    - *Input/Output:* Input from the false branch of `Check Content Quality Status`. True branch goes to `OpenAI Regenerate Captions`; false branch goes to `Compile Failed Report`.
  - **OpenAI Regenerate Captions**
    - *Type and Role:* `@n8n/n8n-nodes-langchain.openAi` (AI / LLM Integration). Fixes previously identified quality issues.
    - *Configuration:* Uses model `gpt-5-mini`. Prompt receives the detected quality issues via `JSON.stringify($json.quality_issues)`, original content, and parameters, instructing the model to fix errors while preserving valid parts.
    - *Input/Output:* Input from the true branch of `Check Retry Limit`; output loops back to `Parse Captions Output`.
    - *Edge Cases:* Repeated generation failures if prompts consistently violate model constraints.
  - **Compile Failed Report**
    - *Type and Role:* `n8n-nodes-base.code` (Data Transformation / JavaScript). Summarizes unresolvable failures.
    - *Configuration:* Generates a failure payload containing accumulated quality issues, status `failed_after_retries`, and a troubleshooting recommendation.
    - *Input/Output:* Input from the false branch of `Check Retry Limit`; output to `Format Telegram Message`.

#### Block 1.5: Telegram Response Dispatch
- **Overview:** Converts success or failure payloads into a formatted Telegram message string, verifies the existence of a recipient chat ID, and sends the final output back to the user.
- **Nodes Involved:**
  - `Format Telegram Message`
  - `Check Telegram Chat Existence`
  - `Send Content Pack via Telegram`
- **Node Details:**
  - **Format Telegram Message**
    - *Type and Role:* `n8n-nodes-base.code` (Data Transformation / JavaScript). Generates a human-readable markdown message.
    - *Configuration:* Handles branching logic based on `data.status`: builds an error summary report for failed items or a structured multi-platform content preview for successful packs.
    - *Input/Output:* Inputs converge from `Compile Content Report` and `Compile Failed Report`; output to `Check Telegram Chat Existence`.
  - **Check Telegram Chat Existence**
    - *Type and Role:* `n8n-nodes-base.if` (Branching). Validates recipient presence.
    - *Configuration:* Checks that `={{ $json.chat_id }}` is not empty.
    - *Input/Output:* Input from `Format Telegram Message`; output to `Send Content Pack via Telegram`.
  - **Send Content Pack via Telegram**
    - *Type and Role:* `n8n-nodes-base.telegram` (Action). Dispatches the formatted text to the user.
    - *Configuration:* Sends message text from `={{ $json.telegram_message }}` to chat ID `={{ $json.chat_id }}` with `appendAttribution` set to false.
    - *Input/Output:* Input from `Check Telegram Chat Existence`.
    - *Edge Cases:* Telegram API rate limits or blocked bot interactions.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `When Topic Received in Telegram` | `n8n-nodes-base.telegramTrigger` | Captures incoming messages | None | `Set Telegram Input Data` | Receive Telegram topic |
| `Set Telegram Input Data` | `n8n-nodes-base.set` | Normalizes chat ID and text | `When Topic Received in Telegram` | `Check Telegram Topic Validity` | Receive Telegram topic |
| `Check Telegram Topic Validity` | `n8n-nodes-base.if` | Validates minimum topic length | `Set Telegram Input Data` | `Set Content Preferences`, `Send Usage Help via Telegram` | Validate topic input |
| `Send Usage Help via Telegram` | `n8n-nodes-base.telegram` | Sends usage instructions | `Check Telegram Topic Validity` | None | Validate topic input |
| `Set Content Preferences` | `n8n-nodes-base.set` | Injects generation defaults | `Check Telegram Topic Validity` | `OpenAI Generate Captions` | Validate topic input |
| `OpenAI Generate Captions` | `@n8n/n8n-nodes-langchain.openAi` | Generates initial social copy | `Set Content Preferences` | `Parse Captions Output` | Generate and parse captions |
| `Parse Captions Output` | `n8n-nodes-base.code` | Cleans and parses AI JSON | `OpenAI Generate Captions`, `OpenAI Regenerate Captions` | `Analyze Content Quality` | Generate and parse captions |
| `Analyze Content Quality` | `n8n-nodes-base.code` | Scores quality and finds issues | `Parse Captions Output` | `Check Content Quality Status` | Score content quality |
| `Check Content Quality Status` | `n8n-nodes-base.if` | Branches on pass/fail status | `Analyze Content Quality` | `Validate Social Content`, `Check Retry Limit` | Score content quality |
| `Validate Social Content` | `n8n-nodes-base.code` | Enforces final validation rules | `Check Content Quality Status` | `Compile Content Report` | Build success report |
| `Compile Content Report` | `n8n-nodes-base.code` | Structures successful payload | `Validate Social Content` | `Format Telegram Message` | Build success report |
| `Check Retry Limit` | `n8n-nodes-base.if` | Controls maximum retry attempts | `Check Content Quality Status` | `OpenAI Regenerate Captions`, `Compile Failed Report` | Retry or fail generation |
| `OpenAI Regenerate Captions` | `@n8n/n8n-nodes-langchain.openAi` | Regenerates flawed content | `Check Retry Limit` | `Parse Captions Output` | Retry or fail generation |
| `Compile Failed Report` | `n8n-nodes-base.code` | Summarizes exhaustion failures | `Check Retry Limit` | `Format Telegram Message` | Retry or fail generation |
| `Format Telegram Message` | `n8n-nodes-base.code` | Formats final markdown output | `Compile Content Report`, `Compile Failed Report` | `Check Telegram Chat Existence` | Send Telegram response |
| `Check Telegram Chat Existence` | `n8n-nodes-base.if` | Validates presence of chat ID | `Format Telegram Message` | `Send Content Pack via Telegram` | Send Telegram response |
| `Send Content Pack via Telegram` | `n8n-nodes-base.telegram` | Sends final report to Telegram | `Check Telegram Chat Existence` | None | Send Telegram response |

---

### 4. Reproducing the Workflow from Scratch

Follow these sequential steps to manually recreate the workflow in n8n:

1. **Create Trigger Node:**
   - Add a **Telegram Trigger** node named `When Topic Received in Telegram`.
   - Set events/updates to `message`. Connect required Telegram API credentials.
2. **Normalize Input:**
   - Add a **Set** node named `Set Telegram Input Data`.
   - Configure assignments: `topic` = `={{ $json.message.text }}`, `chat_id` = `={{ $json.message.chat.id }}`. Include other fields. Connect input from the Telegram trigger.
3. **Validate Topic:**
   - Add an **IF** node named `Check Telegram Topic Validity`.
   - Add expression condition: `={{ typeof $json.topic === 'string' && $json.topic.trim().length >= 10 && !$json.topic.trim().startsWith('/') }}`. Connect input from `Set Telegram Input Data`.
4. **Setup Usage Help Branch (False Output):**
   - Add a **Telegram** node named `Send Usage Help via Telegram`.
   - Set `Text` to a help message indicating minimum character requirements. Set `Chat ID` to `={{ $json.chat_id }}`. Disable attribution. Connect input from the false output of `Check Telegram Topic Validity`.
5. **Setup Content Preferences (True Output):**
   - Add a **Set** node named `Set Content Preferences`.
   - Assign default values: `tone` (Professional), `target_audience` (General audience), `language` (Same as topic), `call_to_action` (Learn more), `include_emojis` (true), `include_hashtags` (true), `brand_voice` (Clear, natural and engaging). Connect input from the true output of `Check Telegram Topic Validity`.
6. **Configure OpenAI Generation:**
   - Add an **OpenAI** node named `OpenAI Generate Captions`.
   - Configure model `gpt-5-mini` and paste the system prompt instructing JSON output for Instagram, TikTok, YouTube Shorts, and Threads based on input variables. Connect credentials. Connect input from `Set Content Preferences`.
7. **Parse AI Output:**
   - Add a **Code** node named `Parse Captions Output`.
   - Insert JavaScript to strip markdown code blocks, parse JSON, handle errors, and merge with preferences from `Set Content Preferences`. Connect input from `OpenAI Generate Captions`.
8. **Analyze Quality:**
   - Add a **Code** node named `Analyze Content Quality`.
   - Insert JavaScript to evaluate character limits, required fields, hashtag counts, emoji flags, and cross-platform duplication, computing a quality score and identifying issues. Connect input from `Parse Captions Output`.
9. **Check Quality Status:**
   - Add an **IF** node named `Check Content Quality Status`.
   - Set condition to check if `={{ $json.quality_status }}` equals `passed`. Connect input from `Analyze Content Quality`.
10. **Build Success Branch (True Output):**
    - Add a **Code** node named `Validate Social Content` with JavaScript enforcing strict final validation assertions.
    - Add a **Code** node named `Compile Content Report` to structure the payload into a clean hierarchical report. Connect `Validate Social Content` to `Compile Content Report`. Connect input from the true branch of `Check Content Quality Status`.
11. **Build Retry/Failure Branch (False Output):**
    - Add an **IF** node named `Check Retry Limit` evaluating `={{ $runIndex < 2 }}`. Connect input from the false branch of `Check Content Quality Status`.
    - **True Retry Path:** Add an **OpenAI** node named `OpenAI Regenerate Captions` using model `gpt-5-mini` with the regeneration prompt incorporating `{{ $json.quality_issues }}`. Connect its output back to `Parse Captions Output`.
    - **False Exhaustion Path:** Add a **Code** node named `Compile Failed Report` to structure the failure payload. Connect input from the false branch of `Check Retry Limit`.
12. **Format and Dispatch Response:**
    - Add a **Code** node named `Format Telegram Message` to generate human-readable markdown strings for both success and failure reports. Connect inputs from `Compile Content Report` and `Compile Failed Report`.
    - Add an **IF** node named `Check Telegram Chat Existence` verifying `={{ $json.chat_id }}` is not empty. Connect input from `Format Telegram Message`.
    - Add a **Telegram** node named `Send Content Pack via Telegram`. Set `Text` to `={{ $json.telegram_message }}` and `Chat ID` to `={{ $json.chat_id }}`. Disable attribution. Connect input from `Check Telegram Chat Existence`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| This workflow does not automatically publish content to social media platforms; it generates a validated content pack delivered via Telegram. | Operational limitation |
| Users must configure their own Telegram Bot credentials and OpenAI API keys within the respective integration nodes after template import. | Credential requirement |
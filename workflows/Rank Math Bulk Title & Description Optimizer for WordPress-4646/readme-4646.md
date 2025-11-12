Rank Math Bulk Title & Description Optimizer for WordPress

https://n8nworkflows.xyz/workflows/rank-math-bulk-title---description-optimizer-for-wordpress-4646


# Rank Math Bulk Title & Description Optimizer for WordPress

### 1. Workflow Overview

This workflow is designed to bulk optimize WordPress post titles and descriptions using AI-generated suggestions, specifically targeting SEO improvements compatible with Rank Math SEO plugin standards. It automates the retrieval of multiple WordPress posts, decides if their metadata should be rewritten, requests AI-generated optimized titles and descriptions, and updates the WordPress post metadata accordingly.

The workflow is logically divided into these blocks:

- **1.1 Input Reception and Setup:** Manual trigger and initial setting of WordPress site URL and API parameters.
- **1.2 Post Retrieval and Batching:** Fetching post IDs from WordPress and limiting the batch size for processing.
- **1.3 Post Processing Loop:** For each post, fetch post data, decide whether to rewrite metadata, and either skip or proceed to AI optimization.
- **1.4 AI Title & Description Generation:** Using OpenRouter Chat Model and structured output parsing to generate SEO-optimized metadata.
- **1.5 Metadata Update:** Updating WordPress post meta fields with the AI-generated title and description.
- **1.6 Workflow Completion:** Final no-operation node indicating workflow end.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Setup

- **Overview:** This block initiates the workflow manually and sets essential configuration parameters such as the WordPress API endpoint URL.
- **Nodes Involved:**  
  - When clicking ‘Execute workflow’  
  - settings  

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Starts the workflow manually on user demand.  
    - Configuration: No parameters required.  
    - Connections: Outputs to `settings`.  
    - Edge cases: User must manually trigger; no automatic scheduling.
  
  - **settings**  
    - Type: Set node  
    - Role: Holds workflow-wide settings such as the WordPress base URL with a trailing slash (critical for correct API calls).  
    - Configuration: Key-value pairs (not shown explicitly) expected to contain URL like `https://example.com/`.  
    - Connections: Outputs to `Get Posts ID`.  
    - Notes: Sticky note reminds to keep trailing slash in URL.  
    - Edge cases: Missing or incorrect URL formatting will cause HTTP requests to fail.

#### 1.2 Post Retrieval and Batching

- **Overview:** Retrieves all WordPress post IDs via HTTP request, then limits the number of posts processed per workflow execution by batching.
- **Nodes Involved:**  
  - Get Posts ID  
  - Limit  
  - Post (splitInBatches)  

- **Node Details:**

  - **Get Posts ID**  
    - Type: HTTP Request  
    - Role: Calls WordPress REST API to fetch post IDs.  
    - Configuration: Uses the URL from `settings` node to form the API endpoint, typically `/wp-json/wp/v2/posts` or similar.  
    - Input: From `settings`.  
    - Output: List of post IDs to process.  
    - Edge cases: Auth failure, URL incorrect, network timeout, empty post list.  

  - **Limit**  
    - Type: Limit node  
    - Role: Restricts the number of posts processed per run (default set to 5 for testing).  
    - Configuration: Limit count set to 5 (can be disabled for full processing).  
    - Input: From `Get Posts ID`.  
    - Output: Limited subset to `Post`.  
    - Notes: Sticky note clarifies this is for testing only.  
    - Edge cases: Limits may hide underlying issues if not disabled in production.  

  - **Post**  
    - Type: SplitInBatches  
    - Role: Processes posts one by one or in small batches for API rate limiting and ordered processing.  
    - Configuration: Default batch size (not explicitly shown).  
    - Input: From `Limit`.  
    - Output: First to `Finish 🚀`, second to `Get post`.  
    - Edge cases: Batch size too large may cause rate limits or slow processing.

#### 1.3 Post Processing Loop

- **Overview:** For each post ID batch, fetch the full post data from WordPress, evaluate if metadata should be rewritten, then either skip or proceed to AI processing.
- **Nodes Involved:**  
  - Get post  
  - Should I Rewrite  
  - Finish 🚀 (early exit)  

- **Node Details:**

  - **Get post**  
    - Type: WordPress node  
    - Role: Retrieves full post details including current title and metadata.  
    - Configuration: Uses post ID from previous node, WordPress credentials configured.  
    - Input: From `Post`.  
    - Output: To `Should I Rewrite`.  
    - Edge cases: Auth errors, post deleted, invalid ID.

  - **Should I Rewrite**  
    - Type: If node  
    - Role: Decision node checking if metadata needs rewriting (likely based on conditions such as empty meta or outdated content).  
    - Configuration: Expression-based condition (not explicitly shown).  
    - Input: From `Get post`.  
    - Output:  
      - True: Proceed to `Create Meta Infos` (AI processing).  
      - False: Loop back to `Post` to process next batch.  
    - Edge cases: Expression failures, false negatives/positives leading to unnecessary rewrites or skipped optimization.

  - **Finish 🚀**  
    - Type: NoOp  
    - Role: Marks end of processing for a batch or signals completion of workflow.  
    - Input: From `Post` (batch end).  
    - Edge cases: None.

#### 1.4 AI Title & Description Generation

- **Overview:** This block sends post content to an AI language model to generate SEO-optimized titles and descriptions, then parses the structured AI output.
- **Nodes Involved:**  
  - OpenRouter Chat Model  
  - Structured Output Parser  
  - Create Meta Infos  

- **Node Details:**

  - **OpenRouter Chat Model**  
    - Type: Langchain OpenRouter Chat Model node  
    - Role: Sends prompt to OpenRouter AI chat model for metadata generation.  
    - Configuration:  
      - Model parameters (temperature, max tokens, prompt template) set internally.  
      - Uses AI credentials (OpenRouter API key).  
    - Input: From `Create Meta Infos` or as part of chain? Actually, connected as `ai_languageModel` input to `Create Meta Infos`.  
    - Output: To `Create Meta Infos`.  
    - Edge cases: API key invalid, rate limit, timeout, malformed prompts.

  - **Structured Output Parser**  
    - Type: Langchain Output Parser node  
    - Role: Parses AI response into structured JSON for easy field extraction.  
    - Input: Connected as `ai_outputParser` to `Create Meta Infos`.  
    - Output: Parsed metadata fields (title, description).  
    - Edge cases: Parsing errors if AI output deviates from expected format.

  - **Create Meta Infos**  
    - Type: Langchain Chain LLM node  
    - Role: Orchestrates the AI prompt, model call, and output parsing to produce final metadata.  
    - Input: From `Should I Rewrite` (if true) and integrates outputs from OpenRouter Chat Model and Structured Output Parser.  
    - Output: To `Update Post Metas`.  
    - Edge cases: Chain execution failure, missing or incomplete AI output.

#### 1.5 Metadata Update

- **Overview:** Updates the WordPress post metadata with the new title and description generated by AI.
- **Nodes Involved:**  
  - Update Post Metas  
  - Post (loop back)  

- **Node Details:**

  - **Update Post Metas**  
    - Type: HTTP Request  
    - Role: Calls WordPress REST API to update post meta fields for SEO (e.g. Rank Math title and description).  
    - Configuration:  
      - Uses WordPress API endpoint from `settings`.  
      - Auth credentials set for WordPress API.  
      - HTTP method: likely PATCH or POST.  
      - Payload includes AI-generated metadata.  
    - Input: From `Create Meta Infos`.  
    - Output: Loops back to `Post` for next batch processing.  
    - Edge cases: Auth failure, API rejection, network errors, partial updates.

---

### 3. Summary Table

| Node Name                  | Node Type                          | Functional Role                             | Input Node(s)              | Output Node(s)             | Sticky Note                                      |
|----------------------------|----------------------------------|--------------------------------------------|----------------------------|----------------------------|-------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                   | Starts workflow manually                     |                            | settings                   |                                                 |
| settings                   | Set                              | Stores configuration like WordPress URL     | When clicking ‘Execute workflow’ | Get Posts ID               | // IMPORTANT: Don't forget the trailing slash at the end, e.g., https://example.com/ |
| Get Posts ID               | HTTP Request                     | Retrieves post IDs from WordPress            | settings                   | Limit                      |                                                 |
| Limit                      | Limit                            | Restricts number of posts processed per run | Get Posts ID               | Post                       | // TEST: Limits processing to 5 items. Disable/remove for production. |
| Post                       | SplitInBatches                   | Processes posts in batches                    | Limit                      | Finish 🚀, Get post         |                                                 |
| Finish 🚀                  | NoOp                             | Marks workflow or batch completion           | Post                       |                            |                                                 |
| Get post                   | WordPress                        | Retrieves full post data                      | Post                       | Should I Rewrite           |                                                 |
| Should I Rewrite           | If                               | Decides whether to rewrite metadata          | Get post                   | Create Meta Infos (true), Post (false) |                                                 |
| Create Meta Infos          | Langchain Chain LLM              | Generates SEO title and description via AI  | Should I Rewrite (true), OpenRouter Chat Model (ai_languageModel), Structured Output Parser (ai_outputParser) | Update Post Metas          |                                                 |
| OpenRouter Chat Model      | Langchain LM Chat OpenRouter     | AI language model call                        | Create Meta Infos (ai_languageModel) | Create Meta Infos          |                                                 |
| Structured Output Parser   | Langchain Output Parser          | Parses AI response into structured data      | Create Meta Infos (ai_outputParser) | Create Meta Infos          |                                                 |
| Update Post Metas          | HTTP Request                     | Updates WordPress post metadata               | Create Meta Infos          | Post                       |                                                 |
| Sticky Note                | Sticky Note                     |                                                 |                            |                            |                                                 |
| Sticky Note1               | Sticky Note                     |                                                 |                            |                            |                                                 |
| Sticky Note2               | Sticky Note                     |                                                 |                            |                            |                                                 |
| Sticky Note3               | Sticky Note                     |                                                 |                            |                            |                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node**  
   - Name: `When clicking ‘Execute workflow’`  
   - Purpose: Manual workflow start.

2. **Create Set node**  
   - Name: `settings`  
   - Purpose: Store WordPress site URL with trailing slash, e.g., `https://example.com/`  
   - Connect: Output of `When clicking ‘Execute workflow’` to input of `settings`.

3. **Create HTTP Request node**  
   - Name: `Get Posts ID`  
   - Purpose: Fetch post IDs from WordPress REST API endpoint `/wp-json/wp/v2/posts`  
   - Configure URL combining `settings` URL + endpoint.  
   - Connect: Output of `settings` to input of `Get Posts ID`.

4. **Create Limit node**  
   - Name: `Limit`  
   - Purpose: Limit number of posts processed per run (default 5 for testing)  
   - Connect: Output of `Get Posts ID` to input of `Limit`.

5. **Create SplitInBatches node**  
   - Name: `Post`  
   - Purpose: Process posts one by one or in batches.  
   - Connect: Output of `Limit` to input of `Post`.

6. **Create NoOp node**  
   - Name: `Finish 🚀`  
   - Purpose: Marks end of processing for batch or workflow.  
   - Connect: First output of `Post` to input of `Finish 🚀`.

7. **Create WordPress node**  
   - Name: `Get post`  
   - Purpose: Fetch full post data for given post ID from `Post`.  
   - Configure with WordPress credentials (OAuth2 or Basic Auth).  
   - Connect: Second output of `Post` to input of `Get post`.

8. **Create If node**  
   - Name: `Should I Rewrite`  
   - Purpose: Decide if metadata needs rewriting based on conditions (e.g., empty SEO title or description).  
   - Configure expression to check post metadata fields.  
   - Connect: Output of `Get post` to input of `Should I Rewrite`.

9. **Create Langchain Chain LLM node**  
   - Name: `Create Meta Infos`  
   - Purpose: Generate SEO optimized title and description using AI.  
   - Connect: True output of `Should I Rewrite` to input of `Create Meta Infos`.

10. **Create Langchain OpenRouter Chat Model node**  
    - Name: `OpenRouter Chat Model`  
    - Purpose: AI language model call.  
    - Configure with OpenRouter API credentials.  
    - Connect as `ai_languageModel` input to `Create Meta Infos`.

11. **Create Langchain Output Parser node**  
    - Name: `Structured Output Parser`  
    - Purpose: Parse AI output into structured JSON.  
    - Connect as `ai_outputParser` input to `Create Meta Infos`.

12. **Create HTTP Request node**  
    - Name: `Update Post Metas`  
    - Purpose: Update WordPress post meta fields with AI-generated title and description.  
    - Configure WordPress REST API endpoint for post meta update.  
    - Use appropriate HTTP method (PATCH/POST) and authentication.  
    - Connect: Output of `Create Meta Infos` to input of `Update Post Metas`.

13. **Connect output of `Update Post Metas` back to `Post` node**  
    - Purpose: Continue batch processing loop.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                          |
|------------------------------------------------------------------------------------------------|-----------------------------------------|
| // IMPORTANT: Don't forget the trailing slash at the end, e.g., https://example.com/           | Attached to `settings` node              |
| // TEST: Limits processing to 5 items. Disable/remove for production.                           | Attached to `Limit` node                  |
| This workflow requires WordPress REST API credentials configured with permissions to read and update posts and metadata. | Authentication requirement               |
| OpenRouter API credentials must be set up in n8n for AI generation to work.                     | AI integration requirement               |
| The AI prompt and output parser must be carefully configured to ensure structured JSON output for metadata fields. | AI prompt engineering and parsing note  |

---

This complete documentation enables deep understanding, reproduction, and modification of the "Rank Math Bulk Title & Description Optimizer for WordPress" workflow, while anticipating potential error sources and integration issues.
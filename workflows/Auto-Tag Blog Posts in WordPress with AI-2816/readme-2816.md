Auto-Tag Blog Posts in WordPress with AI

https://n8nworkflows.xyz/workflows/auto-tag-blog-posts-in-wordpress-with-ai-2816


# Auto-Tag Blog Posts in WordPress with AI

### 1. Workflow Overview

This workflow automates the process of tagging WordPress blog posts using AI-generated suggestions. It is designed for marketers and content managers aiming to streamline content categorization and improve SEO by automatically generating, verifying, and assigning relevant tags to WordPress posts.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Triggering the workflow via an RSS feed or manual invocation to receive blog post content.
- **1.2 AI Tag Generation:** Using OpenAI models to generate contextually relevant tags based on the post content.
- **1.3 Tag Verification and Creation:** Checking existing WordPress tags, identifying missing tags, and creating new tags in WordPress if necessary.
- **1.4 Post Update:** Updating the WordPress post with the verified and newly created tags.
- **1.5 Workflow Orchestration and Utilities:** Handling batch processing, looping over articles, and managing data transformations.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block initiates the workflow by detecting new blog posts from an RSS feed or manual trigger, preparing the article data for processing.

**Nodes Involved:**  
- RSS Feed Trigger  
- Demo Usage in Another Workflow (Execute Workflow Trigger)  
- Loop over articles (SplitInBatches)  
- SET initial record (Set)  

**Node Details:**

- **RSS Feed Trigger**  
  - Type: Trigger node for RSS feed reading  
  - Configuration: Polls every minute for new RSS feed items  
  - Input: External RSS feed URL (configured outside the workflow)  
  - Output: Emits new blog post items with metadata (title, content, categories, tags)  
  - Failure modes: Network errors, invalid RSS feed format, no new items  
  - Notes: Entry point for automated tagging on new posts  

- **Demo Usage in Another Workflow**  
  - Type: Execute Workflow Trigger  
  - Configuration: Triggers the main workflow for each article batch  
  - Input: Receives batches of articles from upstream workflows  
  - Output: Passes articles to the main workflow for tagging  
  - Failure modes: Workflow ID misconfiguration, execution permission errors  

- **Loop over articles**  
  - Type: SplitInBatches  
  - Configuration: Processes articles one by one or in batches  
  - Input: Array of articles from RSS or manual trigger  
  - Output: Single article per execution cycle  
  - Failure modes: Batch size misconfiguration, empty input arrays  

- **SET initial record**  
  - Type: Set node  
  - Configuration: Passes through all fields, preserving article data for downstream use  
  - Input: Single article JSON  
  - Output: Article JSON with all fields intact  
  - Failure modes: Expression errors if input data is malformed  

---

#### 2.2 AI Tag Generation

**Overview:**  
Generates 3-5 relevant tags for each article using OpenAI language models, applying formatting rules to ensure consistency.

**Nodes Involved:**  
- Generate tags for article (Chain LLM)  
- OpenAI Chat Model (Langchain)  
- OpenAI Chat Model1 (Langchain)  
- Auto-fixing Output Parser (Langchain)  
- Structured Output Parser (Langchain)  

**Node Details:**

- **Generate tags for article**  
  - Type: Chain LLM (Langchain)  
  - Configuration: Prompt instructs AI to provide 3-5 suitable tags in title case based on article content  
  - Input: Article content from RSS feed or manual input  
  - Output: AI-generated tags array  
  - Failure modes: API quota exceeded, malformed prompt, network timeouts  

- **OpenAI Chat Model**  
  - Type: OpenAI Chat Model (Langchain)  
  - Configuration: Uses OpenAI API with configured credentials  
  - Input: Prompt from Generate tags for article node  
  - Output: Raw AI response  
  - Failure modes: Authentication errors, rate limits  

- **OpenAI Chat Model1**  
  - Type: OpenAI Chat Model (Langchain)  
  - Configuration: Secondary AI call for output correction  
  - Input: Output from Structured Output Parser  
  - Output: Refined AI response  
  - Failure modes: Same as above  

- **Auto-fixing Output Parser**  
  - Type: Output Parser (Langchain)  
  - Configuration: Automatically fixes AI output format issues  
  - Input: Raw AI output  
  - Output: Corrected structured data  
  - Failure modes: Parsing errors if AI output is too inconsistent  

- **Structured Output Parser**  
  - Type: Output Parser (Langchain)  
  - Configuration: Parses AI output into JSON schema with tags array  
  - Input: AI response text  
  - Output: JSON object with tags  
  - Failure modes: Schema mismatch, parsing failures  

---

#### 2.3 Tag Verification and Creation

**Overview:**  
Checks existing WordPress tags, identifies missing tags from AI output, creates new tags in WordPress if needed, and compiles the final list of tag IDs for assignment.

**Nodes Involved:**  
- GET WP tags (HTTP Request)  
- Combine slugs (Aggregate)  
- Return missing tags (Code)  
- If (Conditional)  
- Split Out (Split Out)  
- POST WP tags (HTTP Request)  
- GET updated WP tags (HTTP Request)  
- Keep matches (Filter)  
- Combine tag_ids (Aggregate)  

**Node Details:**

- **GET WP tags**  
  - Type: HTTP Request  
  - Configuration: GET request to WordPress REST API endpoint `/wp-json/wp/v2/tags`  
  - Authentication: WordPress API credentials (OAuth2 or API key)  
  - Input: Triggered once per batch  
  - Output: List of existing tags with slugs and IDs  
  - Failure modes: Authentication failure, API downtime, rate limits  

- **Combine slugs**  
  - Type: Aggregate  
  - Configuration: Aggregates all tag slugs from GET WP tags response into an array  
  - Input: Array of tag objects  
  - Output: Array of slugs  
  - Failure modes: Empty input arrays  

- **Return missing tags**  
  - Type: Code (JavaScript)  
  - Configuration: Compares AI-generated tags (normalized to lowercase dash-case) with existing WP tags to find missing ones  
  - Input: AI tags and existing WP tags  
  - Output: Array of missing tags  
  - Failure modes: Expression errors, case mismatch issues (noted in sticky notes)  

- **If**  
  - Type: Conditional  
  - Configuration: Checks if missing tags array is empty  
  - Input: Missing tags array  
  - Output: Branches workflow: create missing tags or proceed with existing tags  
  - Failure modes: Logic errors if input is malformed  

- **Split Out**  
  - Type: Split Out  
  - Configuration: Splits missing tags array to process each missing tag individually  
  - Input: Missing tags array  
  - Output: Single missing tag per execution  
  - Failure modes: Empty arrays, processing delays  

- **POST WP tags**  
  - Type: HTTP Request  
  - Configuration: POST request to WordPress REST API `/wp-json/wp/v2/tags` to create new tags  
  - Query Parameters: `slug` (dash-case), `name` (title case)  
  - Authentication: WordPress API credentials  
  - Input: Single missing tag  
  - Output: Created tag object  
  - Failure modes: Duplicate tags, API errors, authentication failures  

- **GET updated WP tags**  
  - Type: HTTP Request  
  - Configuration: GET request to fetch updated tag list after creation  
  - Authentication: WordPress API credentials  
  - Input: Triggered once after all missing tags are processed  
  - Output: Updated list of tags  
  - Failure modes: Same as GET WP tags  

- **Keep matches**  
  - Type: Filter  
  - Configuration: Filters updated WP tags to keep only those matching the AI-generated tags (normalized)  
  - Input: Updated WP tags and AI tags  
  - Output: Filtered tags relevant to the article  
  - Failure modes: Case sensitivity issues, empty results  

- **Combine tag_ids**  
  - Type: Aggregate  
  - Configuration: Aggregates tag IDs from filtered tags into an array for assignment  
  - Input: Filtered tag objects  
  - Output: Array of tag IDs  
  - Failure modes: Empty input arrays  

---

#### 2.4 Post Update

**Overview:**  
Updates the WordPress post with the final list of tag IDs, completing the tagging automation.

**Nodes Involved:**  
- Return article details (Set)  
- Wordpress (WordPress node)  
- MOCK article (Set)  
- Auto-Tag Posts in WordPress (Execute Workflow)  

**Node Details:**

- **Return article details**  
  - Type: Set  
  - Configuration: Prepares post data including title, content, and tag IDs for WordPress update  
  - Input: Article data and tag IDs  
  - Output: Structured post data for WordPress node  
  - Failure modes: Missing fields, expression errors  

- **Wordpress**  
  - Type: WordPress node  
  - Configuration: Creates or updates a WordPress post with assigned tags  
  - Parameters: Title, content, tags (by ID)  
  - Authentication: WordPress API credentials  
  - Input: Post data from Return article details  
  - Output: Confirmation of post update  
  - Failure modes: Authentication errors, API limits, invalid tag IDs  

- **MOCK article**  
  - Type: Set  
  - Configuration: Creates a mock article object combining RSS feed data and AI-generated tags for testing or demonstration  
  - Input: RSS feed item and AI tags  
  - Output: Mock article JSON  
  - Failure modes: Data mismatch, missing fields  

- **Auto-Tag Posts in WordPress**  
  - Type: Execute Workflow  
  - Configuration: Calls this workflow recursively for each article (mode: each)  
  - Input: Mock article or real article data  
  - Output: Triggers tagging process for each article  
  - Failure modes: Recursive call limits, workflow ID misconfiguration  

---

#### 2.5 Workflow Orchestration and Utilities

**Overview:**  
Manages batch processing, data transformations, and provides notes and instructions for users.

**Nodes Involved:**  
- Sticky Notes (multiple)  
- Return missing tags (Code)  
- Combine slugs (Aggregate)  
- Combine tag_ids (Aggregate)  

**Node Details:**

- **Sticky Notes**  
  - Type: Sticky Note  
  - Configuration: Provide documentation, usage instructions, warnings about case sensitivity, and credits  
  - Input/Output: None (visual aid only)  
  - Notes: Important for understanding workflow nuances and setup  

- **Return missing tags** (also part of tag verification)  
  - See above for details  

- **Combine slugs** and **Combine tag_ids**  
  - See above for details  

---

### 3. Summary Table

| Node Name                   | Node Type                           | Functional Role                          | Input Node(s)                | Output Node(s)               | Sticky Note                                                                                          |
|-----------------------------|-----------------------------------|----------------------------------------|-----------------------------|-----------------------------|----------------------------------------------------------------------------------------------------|
| RSS Feed Trigger            | RSS Feed Read Trigger              | Initiates workflow on new blog posts   | —                           | Generate tags for article    | Demo Usage in Another Workflow (Tagging an article discovered with an RSS feed)                     |
| Demo Usage in Another Workflow | Execute Workflow Trigger          | Triggers main workflow for articles    | —                           | Loop over articles           | Auto-Tag Posts in WordPress: AI handles tagging with no manual data entry                           |
| Loop over articles          | SplitInBatches                    | Processes articles one by one           | Demo Usage in Another Workflow | SET initial record          | One of the few potential failure points: case consistency important when checking tags             |
| SET initial record          | Set                              | Preserves article data                  | Loop over articles           | GET WP tags                 |                                                                                                    |
| GET WP tags                | HTTP Request                     | Retrieves existing WordPress tags       | SET initial record           | Combine slugs               |                                                                                                    |
| Combine slugs              | Aggregate                        | Aggregates tag slugs into array         | GET WP tags                  | Return missing tags         |                                                                                                    |
| Return missing tags        | Code                            | Identifies tags missing from WordPress  | Combine slugs, AI tags       | If                         |                                                                                                    |
| If                        | Conditional                     | Checks if missing tags exist             | Return missing tags          | GET updated WP tags / Split Out | If missing tags, create them; else proceed with existing tags                                     |
| Split Out                 | Split Out                       | Splits missing tags for creation         | If                         | POST WP tags                |                                                                                                    |
| POST WP tags              | HTTP Request                   | Creates missing tags in WordPress        | Split Out                   | GET updated WP tags         |                                                                                                    |
| GET updated WP tags       | HTTP Request                   | Fetches updated tags after creation      | POST WP tags, If            | Keep matches                | What's this? If missing tags, create; else keep relevant tags                                      |
| Keep matches              | Filter                         | Filters tags matching AI-generated tags | GET updated WP tags          | Combine tag_ids             |                                                                                                    |
| Combine tag_ids           | Aggregate                      | Aggregates tag IDs for assignment        | Keep matches                 | Loop over articles          |                                                                                                    |
| Generate tags for article | Chain LLM (Langchain)           | Generates AI tags for article content    | RSS Feed Trigger            | MOCK article                |                                                                                                    |
| OpenAI Chat Model         | OpenAI Chat Model (Langchain)  | AI model call for tag generation          | Generate tags for article   | Generate tags for article   |                                                                                                    |
| OpenAI Chat Model1        | OpenAI Chat Model (Langchain)  | Secondary AI call for output correction  | Structured Output Parser    | Auto-fixing Output Parser   |                                                                                                    |
| Auto-fixing Output Parser | Output Parser (Langchain)       | Fixes AI output formatting                | OpenAI Chat Model1          | Generate tags for article   |                                                                                                    |
| Structured Output Parser  | Output Parser (Langchain)       | Parses AI output into structured JSON    | Auto-fixing Output Parser   | OpenAI Chat Model1          |                                                                                                    |
| MOCK article              | Set                            | Creates mock article combining data       | Generate tags for article   | Auto-Tag Posts in WordPress |                                                                                                    |
| Auto-Tag Posts in WordPress | Execute Workflow               | Recursively processes each article       | MOCK article                | Return article details      | To ensure data can be passed, select "Run Once for Each Item" if executing as subworkflow          |
| Return article details    | Set                            | Prepares post data for WordPress update  | Auto-Tag Posts in WordPress | Wordpress                  |                                                                                                    |
| Wordpress                 | WordPress node                 | Updates WordPress post with tags          | Return article details      | —                           |                                                                                                    |
| Sticky Note1              | Sticky Note                   | Documentation and usage instructions      | —                           | —                           | Demo Usage in Another Workflow (Tagging an article discovered with an RSS feed)                     |
| Sticky Note2              | Sticky Note                   | Workflow description and purpose          | —                           | —                           | Auto-Tag Posts in WordPress: AI handles tagging with no manual data entry                           |
| Sticky Note3              | Sticky Note                   | Benefits of AI-driven tagging              | —                           | —                           | Hand off tagging to AI for autopilot WordPress management                                          |
| Sticky Note4              | Sticky Note                   | Execution note for subworkflow              | —                           | —                           | Select "Run Once for Each Item" when executing subworkflow                                         |
| Sticky Note5              | Sticky Note                   | Explanation of tag creation logic           | —                           | —                           | If missing tags, create in WP; else keep existing tags                                             |
| Sticky Note6              | Sticky Note                   | Warning about case sensitivity in tags      | —                           | —                           | Case consistency critical when comparing tags                                                     |
| Sticky Note7              | Sticky Note                   | Note on tag case formatting                  | —                           | —                           | Update nodes if different tag case formatting desired                                             |
| Sticky Note8              | Sticky Note                   | Challenge suggestion for categories and tags | —                           | —                           | Extend workflow to handle categories with different API endpoints                                |
| Sticky Note9              | Sticky Note                   | Author credit and LinkedIn link              | —                           | —                           | [Find Ludwig Gerdes on LinkedIn](https://www.linkedin.com/in/ludwiggerdes)                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create RSS Feed Trigger node**  
   - Type: RSS Feed Read Trigger  
   - Configure to poll your blog’s RSS feed every minute or desired interval.

2. **Create Generate tags for article node**  
   - Type: Chain LLM (Langchain)  
   - Prompt:  
     ```
     Please provide 3-5 suitable tags for the following article:

     {{ $json.content }}

     Tag Formatting Rules:
     1. Tags should be in title case
     ```
   - Connect RSS Feed Trigger output to this node’s input.  
   - Set OpenAI API credentials.

3. **Add OpenAI Chat Model node**  
   - Type: OpenAI Chat Model (Langchain)  
   - Use the same OpenAI credentials.  
   - Connect to Generate tags for article node’s AI language model input.

4. **Add Structured Output Parser node**  
   - Type: Output Parser (Langchain)  
   - Configure JSON schema example:  
     ```json
     {
       "tags": ["Germany", "Technology", "Workflow Automation"]
     }
     ```
   - Connect OpenAI Chat Model output to this node.

5. **Add OpenAI Chat Model1 node**  
   - Type: OpenAI Chat Model (Langchain)  
   - Connect Structured Output Parser output to this node’s AI language model input.

6. **Add Auto-fixing Output Parser node**  
   - Type: Output Parser (Langchain)  
   - Connect OpenAI Chat Model1 output to this node.

7. **Connect Auto-fixing Output Parser output back to Generate tags for article node’s AI output parser input** to complete the chain.

8. **Create MOCK article node**  
   - Type: Set  
   - Assign fields:  
     - title: `={{ $('RSS Feed Trigger').item.json.title }}`  
     - content: `={{ $('RSS Feed Trigger').item.json.content }}`  
     - categories: `={{ $('RSS Feed Trigger').item.json.categories }}`  
     - tags: `={{ $json.output.tags }}` (from AI output)  
   - Connect Generate tags for article output to MOCK article.

9. **Create Auto-Tag Posts in WordPress node**  
   - Type: Execute Workflow  
   - Set mode to “each” to process articles individually.  
   - Set workflow ID to this workflow’s ID (self-reference).  
   - Connect MOCK article output to this node.

10. **Create SET initial record node**  
    - Type: Set  
    - Include all fields from input.  
    - Connect Auto-Tag Posts in WordPress output to this node.

11. **Create GET WP tags node**  
    - Type: HTTP Request  
    - Method: GET  
    - URL: `https://www.example.com/wp-json/wp/v2/tags` (replace with your WP URL)  
    - Authentication: Use WordPress API credentials (OAuth2 or API key)  
    - Connect SET initial record output to this node.

12. **Create Combine slugs node**  
    - Type: Aggregate  
    - Aggregate field: `slug` from GET WP tags output  
    - Connect GET WP tags output to this node.

13. **Create Return missing tags node**  
    - Type: Code (JavaScript)  
    - Code:  
      ```js
      const new_ary = $('SET initial record').first().json.tags.map(x => x.toLowerCase().replaceAll(" ","-")).filter(x => !$input.first().json.tags.includes(x));
      return {"missing_tags": new_ary};
      ```  
    - Connect Combine slugs output and AI tags to this node.

14. **Create If node**  
    - Type: Conditional  
    - Condition: Check if `missing_tags` array is empty  
    - Connect Return missing tags output to this node.

15. **Create Split Out node**  
    - Type: Split Out  
    - Field to split: `missing_tags`  
    - Destination field: `missing_tag`  
    - Connect If node’s “false” branch (missing tags exist) to this node.

16. **Create POST WP tags node**  
    - Type: HTTP Request  
    - Method: POST  
    - URL: `https://www.example.com/wp-json/wp/v2/tags`  
    - Query parameters:  
      - slug: `={{ $json.missing_tag }}`  
      - name: `={{ $json.missing_tag.replaceAll("-", " ").toTitleCase() }}`  
    - Authentication: WordPress API credentials  
    - Connect Split Out output to this node.

17. **Create GET updated WP tags node**  
    - Type: HTTP Request  
    - Method: GET  
    - URL: `https://www.example.com/wp-json/wp/v2/tags`  
    - Authentication: WordPress API credentials  
    - Connect POST WP tags output and If node’s “true” branch (no missing tags) to this node.

18. **Create Keep matches node**  
    - Type: Filter  
    - Condition: Keep tags where slug is contained in AI-generated tags (normalized)  
    - Connect GET updated WP tags output to this node.

19. **Create Combine tag_ids node**  
    - Type: Aggregate  
    - Aggregate field: `id` from Keep matches output  
    - Connect Keep matches output to this node.

20. **Connect Combine tag_ids output back to Loop over articles node** to continue processing.

21. **Create Return article details node**  
    - Type: Set  
    - Assign fields:  
      - tag_ids: `={{ $json.tag_ids }}`  
      - title: `={{ $('MOCK article').item.json.title }}`  
      - content: `={{ $('MOCK article').item.json.content }}`  
    - Connect Auto-Tag Posts in WordPress output to this node.

22. **Create Wordpress node**  
    - Type: WordPress node  
    - Parameters:  
      - Title: `=Demo tagging post: {{ $json.title }}`  
      - Content: `=This is a post to demo automatic tagging a WordPress post via n8n.\n\n{{ $json.content }}`  
      - Tags: `={{ $json.tag_ids }}` (array of tag IDs)  
    - Authentication: WordPress API credentials  
    - Connect Return article details output to this node.

23. **Add Sticky Notes** as per the original workflow to provide documentation, warnings, and credits.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                  |
|----------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Demo Usage in Another Workflow (Tagging an article discovered with an RSS feed)                           | Sticky Note1 near RSS Feed Trigger and Demo Usage nodes                                        |
| Auto-Tag Posts in WordPress: AI handles tagging with no manual data entry                                | Sticky Note2 near workflow start                                                               |
| Handing off tagging and categorization fully to AI lets you put your WordPress account on autopilot      | Sticky Note3 near tag verification nodes                                                       |
| To ensure data can be passed to subsequent nodes, make sure to select "Run Once for Each Item" if executing a subworkflow | Sticky Note4 near Execute Workflow node                                                        |
| If there are missing tags we create them in WP, otherwise we keep get them all from WP and keep the relevant ones | Sticky Note5 near tag creation logic                                                           |
| One of the few potential failure points in this workflow, when checking for missing tags it's important that both the generated tags and the existing tags are in the same case (snake, dash, title) | Sticky Note6 near Loop over articles and SET initial record                                    |
| If you want your tags to follow a different case than I am using (dash case for slug, title case for name), then you will need to update a few nodes in this workflow | Sticky Note7 near Return missing tags node                                                     |
| Ready for a challenge? Make this subworkflow executable for both categories and tags, accounting for different API calls to different endpoints | Sticky Note8 near Keep matches and Combine tag_ids nodes                                       |
| About the maker: Find Ludwig Gerdes on LinkedIn                                                          | [LinkedIn Profile](https://www.linkedin.com/in/ludwiggerdes) (Sticky Note9 near Wordpress node) |

---

This documentation provides a detailed and structured reference for understanding, reproducing, and modifying the "Auto-Tag Blog Posts in WordPress with AI" workflow, including potential failure points and integration considerations.
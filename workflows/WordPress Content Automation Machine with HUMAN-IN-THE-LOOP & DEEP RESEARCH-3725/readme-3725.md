WordPress Content Automation Machine with HUMAN-IN-THE-LOOP & DEEP RESEARCH

https://n8nworkflows.xyz/workflows/wordpress-content-automation-machine-with-human-in-the-loop---deep-research-3725


# WordPress Content Automation Machine with HUMAN-IN-THE-LOOP & DEEP RESEARCH

### 1. Workflow Overview

This workflow is a comprehensive **Human-in-the-Loop Content Automation Machine** designed for WordPress blog automation with deep research capabilities. It targets content creators, marketing teams, SEO specialists, and agencies who want to leverage AI for content generation while maintaining full editorial control through human oversight. The workflow integrates Airtable as the central management interface, OpenAI and Perplexity for AI-driven research and writing, and WordPress for publishing.

The workflow is logically divided into four main interconnected n8n flows plus a sub-workflow for research:

- **1.1 Topic Initiation & Chapter Generation:** Triggered by new topics in Airtable, performs initial research, generates chapter outlines, and saves them for review.
- **1.2 Content Generation & Internal Linking:** For selected chapters, performs deep research, drafts content incorporating internal links, and saves drafts for human review.
- **1.3 Post Assembly & Image Generation:** Aggregates approved content, generates SEO metadata, titles, tags, and images, converts content to HTML, and prepares the final post.
- **1.4 WordPress Publishing & Backup:** Publishes the finalized post to WordPress, uploads images, and archives post data in Airtable.
- **1.5 Research Sub-Workflow:** Handles external research queries using Perplexity or other research tools, invoked by main flows.

Each flow is triggered by specific status and checkbox changes in Airtable tables, ensuring a controlled, stepwise progression with mandatory human approvals.

---

### 2. Block-by-Block Analysis

#### 2.1 Topic Initiation & Chapter Generation (Flow 1)

- **Overview:**  
  This block monitors the “Create Topics” Airtable table for new topics marked "To Do" and triggers research and chapter outline generation. It fetches global settings, performs initial AI research, generates chapter suggestions, and writes them back to Airtable for review.

- **Nodes Involved:**  
  - Create New Topic (Airtable Trigger)  
  - Airtable Get Topic  
  - Split Out All Records  
  - Filter Status  
  - Limit To First Item  
  - Check Status & Confirmation (If)  
  - Confirm Topic (Airtable)  
  - Settings (Code)  
  - Airtable Settings  
  - Check Inputs (If)  
  - Initial Research (Langchain Agent)  
  - OpenAI Chat Model  
  - Structured Output Parser4  
  - Parse Chapters (Code)  
  - Split Out  
  - Airtable Generate Chapters  
  - Split Chapter IDs  
  - Loop Over Chapters  
  - Airtable Generate Chapters - Delete Record  
  - Wait To Delete Chapter  
  - Wait For AI APIs  
  - Airtable Select Chapters - Create Record  
  - Wait To Create Record  
  - Airtable Create Topic - Update Status

- **Node Details:**  
  - **Create New Topic (Airtable Trigger):** Watches the “Create Topics” table for new records with 'Status' = "To Do" and 'Execute Flow' checked. Triggers the flow.  
  - **Airtable Get Topic:** Retrieves detailed data for the triggered topic record.  
  - **Split Out All Records:** Splits multiple records for individual processing.  
  - **Filter Status:** Filters records to only those with the correct status and execution flag.  
  - **Limit To First Item:** Ensures only one topic is processed at a time to avoid API rate limits and concurrency issues.  
  - **Check Status & Confirmation (If):** Verifies the topic is ready for processing.  
  - **Confirm Topic (Airtable):** Updates Airtable to confirm topic processing has started.  
  - **Settings (Code) & Airtable Settings:** Fetches global configuration parameters from the “Settings” table, such as website details, audience, style, and category IDs.  
  - **Check Inputs (If):** Validates required inputs before proceeding.  
  - **Initial Research (Langchain Agent):** Performs AI-driven initial research using Perplexity or other configured research tools.  
  - **OpenAI Chat Model:** Uses OpenAI GPT-4o for generating chapter outlines based on research and settings.  
  - **Structured Output Parser4:** Parses AI output into structured data for chapters.  
  - **Parse Chapters (Code):** Processes parsed data into Airtable-compatible format.  
  - **Split Out:** Splits chapter data for batch processing.  
  - **Airtable Generate Chapters:** Writes generated chapter outlines into the “Generate Chapters” table.  
  - **Split Chapter IDs & Loop Over Chapters:** Processes chapters in batches for deletion or updates.  
  - **Airtable Generate Chapters - Delete Record:** Deletes obsolete chapter records if needed.  
  - **Wait To Delete Chapter:** Wait node to handle API rate limits before deletion.  
  - **Wait For AI APIs:** Wait node to manage rate limits before creating new chapter records.  
  - **Airtable Select Chapters - Create Record:** Creates new chapter records for user selection.  
  - **Wait To Create Record:** Wait node to ensure Airtable API stability.  
  - **Airtable Create Topic - Update Status:** Updates the topic status in Airtable to reflect progress.

- **Edge Cases & Potential Failures:**  
  - Airtable API rate limits causing delays or failures.  
  - AI model timeouts or unexpected output format causing parsing errors.  
  - Missing or incorrect settings leading to incomplete or invalid research prompts.  
  - Multiple topics triggering simultaneously causing concurrency issues (mitigated by limiting to first item).  
  - Network or authentication errors with Airtable or AI APIs.

---

#### 2.2 Content Generation & Internal Linking (Flow 2)

- **Overview:**  
  This block handles selected chapters marked "To Do" in the “Select Chapters” table. It fetches chapter details and global settings, retrieves internal links from the website sitemap, performs deep research per chapter, drafts content incorporating research and internal links, and saves drafts to the “Generate Content” table for review.

- **Nodes Involved:**  
  - Select Chapters Trigger (Airtable Trigger)  
  - Limit To First Item1  
  - Airtable Select Chapters  
  - Split Chapter Process IDs  
  - Loop Over Chapters Process  
  - Airtable Select Chapters - Delete Record  
  - Wait To Delete Chapters Process  
  - Airtable Settings1  
  - Settings1 (Code)  
  - Get Post Sitemap (HTTP Request)  
  - Get XML File (XML Parser)  
  - Split Out Links  
  - Limit Inbound Links  
  - Aggregate internal links  
  - Airtable Chapters1  
  - Parse Chapters (Code)  
  - Split Out  
  - Research Tool1 (Sub-Workflow)  
  - Researcher (Langchain Agent)  
  - OpenAI Chat Model1  
  - Copywriter (Langchain Chain LLM)  
  - Structured Output Parser  
  - Airtable Generate Content1  
  - Wait To Save Content  
  - Loop Over Content  
  - Airtable Generate Content - Delete Record  
  - Wait To Delete Generate Content  
  - Airtable Select Content  
  - Split Content Process IDs  
  - Loop Over Content Process  
  - Airtable Select Content - Delete Record  
  - Wait To Delete Content Process  
  - Airtable Select Content - Update Status  
  - Wait To Update Data  
  - Airtable Select Chapters Update

- **Node Details:**  
  - **Select Chapters Trigger:** Watches “Select Chapters” table for chapters marked "To Do" and 'Execute Flow' checked.  
  - **Limit To First Item1:** Processes one chapter at a time to manage API limits.  
  - **Airtable Select Chapters:** Retrieves chapter details for processing.  
  - **Split Chapter Process IDs & Loop Over Chapters Process:** Processes chapters in batches for deletion or update.  
  - **Airtable Select Chapters - Delete Record:** Deletes obsolete chapter selection records.  
  - **Wait To Delete Chapters Process:** Wait node for rate limit management.  
  - **Airtable Settings1 & Settings1 (Code):** Fetches global settings for content generation.  
  - **Get Post Sitemap (HTTP Request):** Retrieves website sitemap XML to gather internal links.  
  - **Get XML File:** Parses sitemap XML.  
  - **Split Out Links:** Splits individual URLs from sitemap.  
  - **Limit Inbound Links:** Limits the number of internal links to a manageable count.  
  - **Aggregate internal links:** Aggregates selected internal links for use in content.  
  - **Airtable Chapters1:** Retrieves chapter data for content generation.  
  - **Parse Chapters (Code):** Prepares chapter data for AI input.  
  - **Research Tool1 (Sub-Workflow):** Invokes the research sub-workflow to gather deep research data per chapter.  
  - **Researcher (Langchain Agent):** Performs AI research using Perplexity or configured tool.  
  - **OpenAI Chat Model1:** Uses OpenAI GPT-4o for content drafting.  
  - **Copywriter (Langchain Chain LLM):** Refines draft content with AI.  
  - **Structured Output Parser:** Parses AI output into structured content.  
  - **Airtable Generate Content1:** Saves draft content to “Generate Content” table.  
  - **Wait To Save Content:** Wait node for API rate limit management.  
  - **Loop Over Content:** Processes content records for deletion or update.  
  - **Airtable Generate Content - Delete Record:** Deletes obsolete content drafts.  
  - **Wait To Delete Generate Content:** Wait node for rate limit management.  
  - **Airtable Select Content:** Retrieves content for review.  
  - **Split Content Process IDs & Loop Over Content Process:** Processes content in batches.  
  - **Airtable Select Content - Delete Record:** Deletes obsolete content records.  
  - **Wait To Delete Content Process:** Wait node for rate limit management.  
  - **Airtable Select Content - Update Status:** Updates content status after review.  
  - **Wait To Update Data:** Wait node to ensure Airtable updates propagate.  
  - **Airtable Select Chapters Update:** Updates chapter selection status.

- **Edge Cases & Potential Failures:**  
  - Sitemap retrieval failure or malformed XML causing internal link extraction issues.  
  - Research API rate limits or failures.  
  - AI model generating incomplete or irrelevant content.  
  - Airtable API limits causing delays or failures.  
  - User failing to approve or update content status, blocking workflow progression.

---

#### 2.3 Post Assembly & Image Generation (Flow 3)

- **Overview:**  
  This block aggregates approved chapter content, generates the article title, SEO metadata, tags, and images (chapter images and featured image), converts content to HTML, and prepares the final post record in Airtable for review and publishing.

- **Nodes Involved:**  
  - Airtable Select Content Trigger  
  - Check Status And Confirmation (If)  
  - Airtable Select Content2  
  - Limit To First Item2  
  - Airtable Generated Content  
  - New Chapters (Code)  
  - Blog Planner (Langchain Chain LLM)  
  - Structured Output Parser1  
  - Settings2 (Code)  
  - Airtable Settings2  
  - Split Out chapters  
  - Loop Over Chapters2  
  - Generate Chapter Image (OpenAI)  
  - Resize Chapter Image (Edit Image)  
  - Upload Chapter Image (HTTP Request)  
  - Upload Chapter Image To Drive (Google Drive)  
  - Merge Chapter Image Data  
  - Wait To Upload Image  
  - Combine Into Article (Code)  
  - Final Article In Markdown (Set)  
  - Markdown To HTML  
  - FInal Article In HTML (Set)  
  - Merge Article And Feature Image  
  - Generate featured image (OpenAI)  
  - Resize featured image (Edit Image)  
  - Upload featured image to Drive (Google Drive)  
  - Upload Featured Image (HTTP Request)  
  - Update Featured Image Meta Data (HTTP Request)  
  - Airtable Finalize Post  
  - Split Post IDs  
  - Loop Over Records  
  - Airtable Finalize Post - Delete Record  
  - Wait To Delete Post  
  - Airtable Finalize Post1  
  - Wait To Save Post  
  - Airtable Select Content - Update Status

- **Node Details:**  
  - **Airtable Select Content Trigger:** Watches “Select Content” table for approved content marked "To Do" and 'Execute Flow' checked.  
  - **Check Status And Confirmation (If):** Validates readiness for final assembly.  
  - **Airtable Select Content2:** Retrieves content records for processing.  
  - **Limit To First Item2:** Processes one topic at a time.  
  - **Airtable Generated Content:** Fetches generated content for all chapters.  
  - **New Chapters (Code):** Processes chapter data for article assembly.  
  - **Blog Planner (Langchain Chain LLM):** Generates article title, SEO metadata, and tags using AI.  
  - **Structured Output Parser1:** Parses blog planner output.  
  - **Settings2 (Code) & Airtable Settings2:** Fetches global settings for final assembly.  
  - **Split Out chapters & Loop Over Chapters2:** Processes chapters for image generation and article assembly.  
  - **Generate Chapter Image:** Uses OpenAI DALL-E or alternative AI to create chapter images.  
  - **Resize Chapter Image:** Adjusts image size for web use.  
  - **Upload Chapter Image:** Uploads images to WordPress or other storage.  
  - **Upload Chapter Image To Drive:** Optionally uploads images to Google Drive for backup.  
  - **Merge Chapter Image Data:** Combines image metadata with chapter data.  
  - **Wait To Upload Image:** Wait node for API rate limit management.  
  - **Combine Into Article (Code):** Merges chapter content and images into a complete article in Markdown.  
  - **Final Article In Markdown (Set):** Stores combined article in Markdown format.  
  - **Markdown To HTML:** Converts Markdown to HTML for WordPress.  
  - **FInal Article In HTML (Set):** Prepares final HTML content.  
  - **Merge Article And Feature Image:** Combines article content with featured image data.  
  - **Generate featured image:** Creates a featured image for the article.  
  - **Resize featured image:** Adjusts featured image size.  
  - **Upload featured image to Drive:** Optionally uploads featured image to Google Drive.  
  - **Upload Featured Image:** Uploads featured image to WordPress.  
  - **Update Featured Image Meta Data:** Updates WordPress post metadata with featured image info.  
  - **Airtable Finalize Post:** Writes final post data to “Finalize Post” table.  
  - **Split Post IDs & Loop Over Records:** Processes posts for deletion or update.  
  - **Airtable Finalize Post - Delete Record:** Deletes obsolete finalize post records.  
  - **Wait To Delete Post:** Wait node for API rate limit management.  
  - **Airtable Finalize Post1:** Updates finalize post status.  
  - **Wait To Save Post:** Wait node to ensure Airtable updates propagate.  
  - **Airtable Select Content - Update Status:** Updates content status after final assembly.

- **Edge Cases & Potential Failures:**  
  - AI image generation producing low-quality or irrelevant images.  
  - WordPress media upload failures or API errors.  
  - Markdown to HTML conversion errors.  
  - Airtable API limits causing delays.  
  - Missing or incomplete content blocking final assembly.

---

#### 2.4 WordPress Publishing & Backup (Flow 4)

- **Overview:**  
  This block publishes the finalized post to WordPress, uploads images, sets metadata and tags, and creates a backup record in Airtable with the live URL. It ensures the post is published with all necessary SEO and media assets.

- **Nodes Involved:**  
  - Airtable Finalize Post Trigger  
  - Airtable Settings3  
  - Settings3 (Code)  
  - Split Out Records  
  - Filter Post Status And Confirmation  
  - Limit To First Item3  
  - Post on Wordpress (WordPress Node)  
  - Set featured image for post (HTTP Request)  
  - Set excerpt (HTTP Request)  
  - Airtable Backup Post  
  - Wait To Backup Post  
  - Airtable Finalize Post - Update Status  
  - Airtable Finalize Post2  
  - Split Out Records  
  - Airtable Finalize Post - Delete Record  
  - Wait To Delete Post

- **Node Details:**  
  - **Airtable Finalize Post Trigger:** Watches “Finalize Post” table for records marked "To Do" and 'Post to Website' checked.  
  - **Airtable Settings3 & Settings3 (Code):** Fetches global settings for publishing.  
  - **Split Out Records:** Splits multiple posts for individual processing.  
  - **Filter Post Status And Confirmation:** Filters posts ready for publishing.  
  - **Limit To First Item3:** Processes one post at a time.  
  - **Post on Wordpress:** Creates or updates the WordPress post with content, metadata, tags, and status "published".  
  - **Set featured image for post:** Associates the featured image with the WordPress post.  
  - **Set excerpt:** Sets the post excerpt in WordPress.  
  - **Airtable Backup Post:** Creates a backup record in Airtable with post details and live URL.  
  - **Wait To Backup Post:** Wait node for API rate limit management.  
  - **Airtable Finalize Post - Update Status:** Updates post status in Airtable to reflect publishing completion.  
  - **Airtable Finalize Post2:** Retrieves post records for cleanup.  
  - **Split Out Records:** Splits records for deletion.  
  - **Airtable Finalize Post - Delete Record:** Deletes obsolete finalize post records.  
  - **Wait To Delete Post:** Wait node to manage API limits.

- **Edge Cases & Potential Failures:**  
  - WordPress API authentication or permission errors.  
  - Media upload failures causing missing images.  
  - Airtable API limits or update failures.  
  - Network issues causing incomplete publishing.  
  - Incorrect post metadata or tags causing SEO issues.

---

#### 2.5 Research Sub-Workflow

- **Overview:**  
  This sub-workflow is invoked by main flows to perform external research queries using Perplexity or other configured research tools. It returns structured research data for use in AI content generation.

- **Nodes Involved:**  
  - Research Tool (Langchain Tool Workflow)  
  - PerplexityAI API (HTTP Request)  
  - Get Research Content (Set)  
  - Edit Fields (Set)  

- **Node Details:**  
  - **Research Tool:** Acts as an interface node to call the sub-workflow from main workflows.  
  - **PerplexityAI API:** Sends research queries to Perplexity API, retrieving fresh and relevant data.  
  - **Get Research Content:** Processes API response to extract useful content.  
  - **Edit Fields:** Adjusts or formats fields for downstream use.

- **Edge Cases & Potential Failures:**  
  - API rate limits or quota exceeded errors.  
  - Unexpected API response formats causing parsing errors.  
  - Network or authentication failures.

---

### 3. Summary Table

| Node Name                          | Node Type                         | Functional Role                          | Input Node(s)                          | Output Node(s)                         | Sticky Note                                                                                  |
|-----------------------------------|----------------------------------|----------------------------------------|--------------------------------------|--------------------------------------|----------------------------------------------------------------------------------------------|
| Create New Topic                  | Airtable Trigger                 | Trigger flow on new topic               | -                                    | Airtable Get Topic                   |                                                                                              |
| Airtable Get Topic               | Airtable                        | Fetch topic details                     | Create New Topic                     | Split Out All Records                |                                                                                              |
| Split Out All Records            | Split Out                      | Split records for processing            | Airtable Get Topic                   | Filter Status                       |                                                                                              |
| Filter Status                   | Filter                         | Filter records by status                 | Split Out All Records                | Limit To First Item                 |                                                                                              |
| Limit To First Item             | Limit                         | Limit processing to one topic            | Filter Status                       | Check Status & Confirmation         |                                                                                              |
| Check Status & Confirmation     | If                            | Verify topic readiness                    | Limit To First Item                 | Confirm Topic                      |                                                                                              |
| Confirm Topic                  | Airtable                        | Update topic status                      | Check Status & Confirmation         | Settings                          |                                                                                              |
| Settings                      | Code                           | Fetch global settings                    | Confirm Topic                      | Check Inputs                      |                                                                                              |
| Airtable Settings             | Airtable                        | Retrieve settings from Airtable          | -                                    | Settings                          |                                                                                              |
| Check Inputs                  | If                            | Validate required inputs                  | Settings                          | Initial Research                  |                                                                                              |
| Initial Research              | Langchain Agent                | Perform AI research                       | Check Inputs                      | Wait For AI APIs                  |                                                                                              |
| OpenAI Chat Model             | Langchain LM Chat OpenAI       | Generate chapter outlines                 | Initial Research                  | Structured Output Parser4         |                                                                                              |
| Structured Output Parser4     | Langchain Output Parser        | Parse AI output into structured chapters | OpenAI Chat Model                 | Parse Chapters                   |                                                                                              |
| Parse Chapters               | Code                           | Format chapters for Airtable              | Structured Output Parser4          | Split Out                       |                                                                                              |
| Split Out                   | Split Out                      | Split chapters for batch processing       | Parse Chapters                   | Airtable Generate Chapters       |                                                                                              |
| Airtable Generate Chapters    | Airtable                        | Save chapter outlines                     | Split Out                       | Split Chapter IDs                |                                                                                              |
| Split Chapter IDs            | Split Out                      | Split chapter IDs for processing          | Airtable Generate Chapters        | Loop Over Chapters              |                                                                                              |
| Loop Over Chapters           | Split In Batches               | Process chapters in batches                | Split Chapter IDs                | Airtable Generate Chapters - Delete Record |                                                                                              |
| Airtable Generate Chapters - Delete Record | Airtable                        | Delete obsolete chapter records            | Loop Over Chapters              | Wait To Delete Chapter          |                                                                                              |
| Wait To Delete Chapter       | Wait                          | Wait to manage API rate limits             | Airtable Generate Chapters - Delete Record | Loop Over Chapters              |                                                                                              |
| Wait For AI APIs             | Wait                          | Wait to manage AI API rate limits          | Initial Research                  | Split Out Chapters, Airtable Select Chapters - Create Record |                                                                                              |
| Airtable Select Chapters - Create Record | Airtable                        | Create chapter selection records           | Wait For AI APIs                 | Wait To Create Record           |                                                                                              |
| Wait To Create Record        | Wait                          | Wait to ensure Airtable API stability      | Airtable Select Chapters - Create Record | Airtable Create Topic - Update Status |                                                                                              |
| Airtable Create Topic - Update Status | Airtable                        | Update topic status after chapter creation | Wait To Create Record           | -                                |                                                                                              |
| Select Chapters Trigger      | Airtable Trigger               | Trigger flow on selected chapters          | -                                | Limit To First Item1            |                                                                                              |
| Limit To First Item1         | Limit                         | Limit processing to one chapter             | Select Chapters Trigger          | Airtable Select Chapters        |                                                                                              |
| Airtable Select Chapters     | Airtable                        | Fetch selected chapters                     | Limit To First Item1             | Split Chapter Process IDs       |                                                                                              |
| Split Chapter Process IDs    | Split Out                      | Split chapter IDs for processing             | Airtable Select Chapters         | Loop Over Chapters Process      |                                                                                              |
| Loop Over Chapters Process   | Split In Batches              | Process chapters in batches                   | Split Chapter Process IDs        | Airtable Select Chapters - Delete Record |                                                                                              |
| Airtable Select Chapters - Delete Record | Airtable                        | Delete obsolete chapter selection records    | Loop Over Chapters Process       | Wait To Delete Chapters Process |                                                                                              |
| Wait To Delete Chapters Process | Wait                          | Wait to manage API rate limits               | Airtable Select Chapters - Delete Record | Loop Over Chapters Process      |                                                                                              |
| Airtable Settings1           | Airtable                        | Fetch global settings for content generation | -                                | Settings1                      |                                                                                              |
| Settings1                   | Code                           | Process settings for content generation       | Airtable Settings1              | Get Post Sitemap               |                                                                                              |
| Get Post Sitemap             | HTTP Request                  | Retrieve website sitemap XML                   | Settings1                      | Get XML File                  |                                                                                              |
| Get XML File                | XML Parser                   | Parse sitemap XML                              | Get Post Sitemap               | Split Out Links              |                                                                                              |
| Split Out Links             | Split Out                    | Split URLs from sitemap                         | Get XML File                  | Limit Inbound Links          |                                                                                              |
| Limit Inbound Links         | Limit                       | Limit number of internal links                   | Split Out Links               | Aggregate internal links    |                                                                                              |
| Aggregate internal links    | Aggregate                   | Aggregate internal links for content             | Limit Inbound Links           | Airtable Chapters1          |                                                                                              |
| Airtable Chapters1          | Airtable                    | Fetch chapter data for content generation        | Aggregate internal links      | Parse Chapters             |                                                                                              |
| Parse Chapters             | Code                       | Prepare chapter data for AI input                  | Airtable Chapters1            | Research Tool1             |                                                                                              |
| Research Tool1             | Langchain Tool Workflow      | Invoke research sub-workflow                        | Parse Chapters               | Researcher                |                                                                                              |
| Researcher                 | Langchain Agent            | Perform AI research per chapter                      | Research Tool1               | OpenAI Chat Model1        |                                                                                              |
| OpenAI Chat Model1         | Langchain LM Chat OpenAI     | Draft content for chapter                            | Researcher                  | Copywriter               |                                                                                              |
| Copywriter                | Langchain Chain LLM          | Refine draft content                                 | OpenAI Chat Model1          | Structured Output Parser  |                                                                                              |
| Structured Output Parser   | Langchain Output Parser      | Parse AI output into structured content              | Copywriter                 | Airtable Generate Content1 |                                                                                              |
| Airtable Generate Content1 | Airtable                    | Save draft content to Airtable                         | Structured Output Parser     | Wait To Save Content      |                                                                                              |
| Wait To Save Content       | Wait                        | Wait to manage API rate limits                         | Airtable Generate Content1   | Loop Over Content         |                                                                                              |
| Loop Over Content          | Split In Batches            | Process content records in batches                      | Wait To Save Content         | Airtable Generate Content - Delete Record |                                                                                              |
| Airtable Generate Content - Delete Record | Airtable                    | Delete obsolete content drafts                           | Loop Over Content           | Wait To Delete Generate Content |                                                                                              |
| Wait To Delete Generate Content | Wait                        | Wait to manage API rate limits                         | Airtable Generate Content - Delete Record | Loop Over Content          |                                                                                              |
| Airtable Select Content    | Airtable                    | Fetch content for review                                | -                            | Split Content Process IDs |                                                                                              |
| Split Content Process IDs  | Split Out                  | Split content IDs for processing                         | Airtable Select Content      | Loop Over Content Process |                                                                                              |
| Loop Over Content Process  | Split In Batches          | Process content in batches                                | Split Content Process IDs    | Airtable Select Content - Delete Record |                                                                                              |
| Airtable Select Content - Delete Record | Airtable                    | Delete obsolete content records                           | Loop Over Content Process    | Wait To Delete Content Process |                                                                                              |
| Wait To Delete Content Process | Wait                        | Wait to manage API rate limits                         | Airtable Select Content - Delete Record | Loop Over Content Process |                                                                                              |
| Airtable Select Content - Update Status | Airtable                    | Update content status after review                       | Wait To Save Post            | -                          |                                                                                              |
| Wait To Update Data       | Wait                        | Wait to ensure Airtable updates propagate                | Airtable Select Content1     | Airtable Select Chapters Update |                                                                                              |
| Airtable Select Chapters Update | Airtable                    | Update chapter selection status                          | Wait To Update Data          | -                          |                                                                                              |
| Airtable Select Content Trigger | Airtable Trigger           | Trigger flow on content approval                          | -                            | Check Status And Confirmation |                                                                                              |
| Check Status And Confirmation | If                          | Verify content approval status                            | Airtable Select Content Trigger | Airtable Select Content2   |                                                                                              |
| Airtable Select Content2   | Airtable                    | Fetch content records for final assembly                   | Check Status And Confirmation | Limit To First Item2       |                                                                                              |
| Limit To First Item2       | Limit                       | Limit processing to one topic                              | Airtable Select Content2     | Airtable Generated Content |                                                                                              |
| Airtable Generated Content | Airtable                    | Fetch generated content for assembly                       | Limit To First Item2         | New Chapters               |                                                                                              |
| New Chapters              | Code                       | Process chapter content for article assembly               | Airtable Generated Content  | Blog Planner               |                                                                                              |
| Blog Planner              | Langchain Chain LLM          | Generate title, SEO metadata, and tags                      | New Chapters                | Check Empty Output         |                                                                                              |
| Structured Output Parser1 | Langchain Output Parser      | Parse blog planner output                                   | Blog Planner                | Get Planner Output         |                                                                                              |
| Settings2                 | Code                       | Fetch settings for final assembly                            | Airtable Settings2          | Airtable Select Chapters2  |                                                                                              |
| Airtable Settings2        | Airtable                    | Retrieve global settings                                     | -                          | Settings2                  |                                                                                              |
| Split Out chapters        | Split Out                  | Split chapters for image generation and assembly            | Get Planner Output          | Loop Over Chapters2        |                                                                                              |
| Loop Over Chapters2       | Split In Batches          | Process chapters for image generation and assembly          | Split Out chapters          | Generate Chapter Image, Combine Into Article |                                                                                              |
| Generate Chapter Image    | OpenAI                      | Generate AI images for chapters                              | Loop Over Chapters2         | Resize Chapter Image       |                                                                                              |
| Resize Chapter Image      | Edit Image                 | Resize chapter images                                        | Generate Chapter Image      | Upload Chapter Image, Upload Chapter Image To Drive |                                                                                              |
| Upload Chapter Image      | HTTP Request               | Upload chapter images to WordPress or storage                | Resize Chapter Image        | Update Feature Image Meta Data |                                                                                              |
| Upload Chapter Image To Drive | Google Drive               | Upload chapter images to Google Drive for backup             | Resize Chapter Image        | Merge Chapter Image Data   |                                                                                              |
| Merge Chapter Image Data  | Merge                      | Combine image metadata with chapter data                      | Upload Chapter Image, Upload Chapter Image To Drive | Wait To Upload Image        |                                                                                              |
| Wait To Upload Image      | Wait                       | Wait to manage API rate limits                               | Merge Chapter Image Data    | Loop Over Chapters2        |                                                                                              |
| Combine Into Article      | Code                       | Merge chapter content and images into full article            | Loop Over Chapters2         | Final Article In Markdown  |                                                                                              |
| Final Article In Markdown | Set                        | Store article in Markdown format                              | Combine Into Article        | Markdown To HTML           |                                                                                              |
| Markdown To HTML          | Markdown                   | Convert Markdown to HTML                                      | Final Article In Markdown   | FInal Article In HTML      |                                                                                              |
| FInal Article In HTML     | Set                        | Prepare final HTML content                                    | Markdown To HTML            | Merge Article And Feature Image |                                                                                              |
| Merge Article And Feature Image | Merge                      | Combine article content with featured image data              | FInal Article In HTML, Upload featured image to Drive | Upload Featured Image       |                                                                                              |
| Generate featured image   | OpenAI                     | Generate featured image for article                           | Get Planner Output          | Resize featured image      |                                                                                              |
| Resize featured image     | Edit Image                 | Resize featured image                                        | Generate featured image     | Merge3                     |                                                                                              |
| Upload featured image to Drive | Google Drive               | Upload featured image to Google Drive for backup             | Resize featured image       | Merge3                     |                                                                                              |
| Upload Featured Image     | HTTP Request               | Upload featured image to WordPress                            | Merge Article And Feature Image | Update Featured Image Meta Data |                                                                                              |
| Update Featured Image Meta Data | HTTP Request               | Update WordPress post metadata with featured image info       | Upload Featured Image       | Airtable Finalize Post1    |                                                                                              |
| Airtable Finalize Post    | Airtable                   | Save final post data for review and publishing                | Merge3                      | Split Post IDs             |                                                                                              |
| Split Post IDs            | Split Out                  | Split post records for processing                             | Airtable Finalize Post      | Loop Over Records          |                                                                                              |
| Loop Over Records         | Split In Batches           | Process posts in batches                                      | Split Post IDs              | Airtable Finalize Post - Delete Record |                                                                                              |
| Airtable Finalize Post - Delete Record | Airtable                   | Delete obsolete finalize post records                         | Loop Over Records           | Wait To Delete Post        |                                                                                              |
| Wait To Delete Post       | Wait                       | Wait to manage API rate limits                               | Airtable Finalize Post - Delete Record | Loop Over Records          |                                                                                              |
| Airtable Finalize Post1   | Airtable                   | Update finalize post status                                   | Update Featured Image Meta Data | Wait To Save Post          |                                                                                              |
| Wait To Save Post         | Wait                       | Wait to ensure Airtable updates propagate                    | Airtable Finalize Post1     | Airtable Select Content - Update Status |                                                                                              |
| Airtable Select Content - Update Status | Airtable                   | Update content status after final assembly                   | Wait To Save Post           | -                          |                                                                                              |
| Airtable Finalize Post Trigger | Airtable Trigger           | Trigger flow on finalize post                                 | -                          | Airtable Settings3         |                                                                                              |
| Airtable Settings3        | Airtable                   | Fetch global settings for publishing                          | -                          | Settings3                  |                                                                                              |
| Settings3                 | Code                      | Process settings for publishing                               | Airtable Settings3          | Split Out Records          |                                                                                              |
| Split Out Records         | Split Out                 | Split post records for publishing                             | Settings3                  | Filter Post Status And Confirmation |                                                                                              |
| Filter Post Status And Confirmation | Filter                    | Filter posts ready for publishing                             | Split Out Records           | Limit To First Item3       |                                                                                              |
| Limit To First Item3       | Limit                     | Limit processing to one post                                  | Filter Post Status And Confirmation | Post on Wordpress          |                                                                                              |
| Post on Wordpress          | WordPress                 | Create or update WordPress post                               | Limit To First Item3         | Set featured image for post |                                                                                              |
| Set featured image for post | HTTP Request              | Associate featured image with WordPress post                 | Post on Wordpress           | Set excerpt                |                                                                                              |
| Set excerpt                | HTTP Request              | Set post excerpt in WordPress                                 | Set featured image for post | Airtable Backup Post       |                                                                                              |
| Airtable Backup Post       | Airtable                  | Create backup record with live URL                            | Set excerpt                 | Wait To Backup Post        |                                                                                              |
| Wait To Backup Post        | Wait                      | Wait to manage API rate limits                                | Airtable Backup Post        | Airtable Finalize Post - Update Status |                                                                                              |
| Airtable Finalize Post - Update Status | Airtable                  | Update post status after publishing                           | Wait To Backup Post         | Airtable Finalize Post2    |                                                                                              |
| Airtable Finalize Post2    | Airtable                  | Fetch post records for cleanup                                | Airtable Finalize Post - Update Status | Split Out Records          |                                                                                              |
| Split Out Records         | Split Out                 | Split records for deletion                                    | Airtable Finalize Post2     | Airtable Finalize Post - Delete Record |                                                                                              |
| Airtable Finalize Post - Delete Record | Airtable                  | Delete obsolete post records                                  | Split Out Records           | Wait To Delete Post        |                                                                                              |
| Wait To Delete Post       | Wait                      | Wait to manage API rate limits                                | Airtable Finalize Post - Delete Record | -                          |                                                                                              |
| Research Tool             | Langchain Tool Workflow   | Invoke research sub-workflow                                  | -                          | Initial Research           |                                                                                              |
| PerplexityAI API          | HTTP Request              | Send research queries to Perplexity API                      | Edit Fields                 | Get Research Content       | Disabled by default; used in research sub-workflow                                          |
| Get Research Content      | Set                       | Extract research content from API response                   | PerplexityAI API            | -                         | Disabled by default                                                                         |
| Edit Fields              | Set                       | Format fields for downstream use                              | -                          | PerplexityAI API           | Disabled by default                                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Airtable Base:**  
   - Duplicate the provided Airtable base template with all tables: Settings, Create Topics, Generate Chapters, Select Chapters, Generate Content, Select Content, Finalize Post, Backup Post.  
   - Ensure all field names and table names exactly match those expected by the workflow.

2. **Configure Credentials in n8n:**  
   - Add credentials for Airtable (API key), WordPress (Application Password with REST API enabled), OpenAI API key, Perplexity API key, and Google Drive (OAuth2) for backups.

3. **Import Main Workflow JSON:**  
   - Import the main workflow JSON into n8n and name it appropriately (e.g., "Main Flow - Human-in-the-Loop Content Automation").  
   - Import the research sub-workflow JSON separately.

4. **Assign Credentials:**  
   - Open each node requiring authentication (Airtable, HTTP Request, WordPress, Google Drive, OpenAI) and assign the corresponding credentials.

5. **Configure Airtable Nodes:**  
   - For every Airtable node, set the base ID and table name to match your duplicated Airtable base.  
   - Map fields correctly, ensuring field names correspond exactly.

6. **Link Main Workflow to Research Sub-Workflow:**  
   - In the Research Tool nodes, configure the sub-workflow invocation to call the research sub-workflow.  
   - Ensure input/output parameters are correctly mapped.

7. **Set Global Settings:**  
   - Populate the “Settings” table in Airtable with your website URL, audience description, writing style, category IDs, "About Us" text, and Call To Action (CTA) details.

8. **Configure AI Nodes:**  
   - Set OpenAI nodes to use GPT-4o or preferred model.  
   - Set image generation nodes to use DALL-E or alternative AI image models.  
   - Adjust prompts in AI nodes if desired for tone or style.

9. **Configure Wait Nodes:**  
   - Adjust wait times in Wait nodes to suit your API rate limits and usage patterns.

10. **Test Each Flow Manually:**  
    - Trigger each flow manually by updating Airtable records (e.g., set 'Status' to "To Do" and check 'Execute Flow').  
    - Verify data flows correctly through nodes and outputs are as expected.

11. **Activate Main Workflow:**  
    - Enable the main workflow in n8n.  
    - The sub-workflow remains inactive and is invoked only by the main workflow.

12. **Start Content Creation:**  
    - Add new topics in Airtable “Create Topics” table and progress through the workflow steps by updating statuses and checkboxes as per the process.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Full walkthrough video available on YouTube channel                                                      | https://www.youtube.com/@nminhduc/                                                              |
| Workflow designed for “Human-in-the-Loop” content creation with AI and Airtable as control center        | Description section                                                                              |
| Use OpenAI GPT-4o model for best results; other models may cause reliability issues                      | Tips for Pros                                                                                   |
| PerplexityAI “sonar” model is default for research; “sonar-deep-research” available for deeper research  | Tips for Pros                                                                                   |
| Default AI image generation via OpenAI DALL-E; alternative models like FLUX.1 recommended for better quality | Tips for Pros                                                                                   |
| Workflow includes Wait nodes to manage API rate limits                                                   | Important Considerations                                                                         |
| For fully automated parallel processing, consider alternative workflow: [n8n Content Automation Pro – with DEEP RESEARCH](https://n8n.io/workflows/3041-wordpress-auto-blogging-pro-with-deep-research-content-automation-machine/) | Important Considerations                                                                         |
| Requires exact Airtable base structure; modifying tables or fields requires corresponding node updates   | Airtable Database Explanation                                                                   |
| Recommended to implement an error handling workflow in n8n for production use                            | Tips for Pros                                                                                   |
| Backup feature optionally saves content and images to Google Drive                                       | Requirements and Workflow Description                                                           |

---

This document provides a detailed, structured reference to understand, reproduce, and maintain the "WordPress Content Automation Machine with HUMAN-IN-THE-LOOP & DEEP RESEARCH" workflow in n8n. It covers all nodes, their roles, configurations, and integration points, enabling advanced users and AI agents to work effectively with this complex automation.
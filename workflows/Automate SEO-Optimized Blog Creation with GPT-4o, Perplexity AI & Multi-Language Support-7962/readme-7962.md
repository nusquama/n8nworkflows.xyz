Automate SEO-Optimized Blog Creation with GPT-4o, Perplexity AI & Multi-Language Support

https://n8nworkflows.xyz/workflows/automate-seo-optimized-blog-creation-with-gpt-4o--perplexity-ai---multi-language-support-7962


# Automate SEO-Optimized Blog Creation with GPT-4o, Perplexity AI & Multi-Language Support

### 1. Workflow Overview

This workflow automates the creation of SEO-optimized blog posts using GPT-4o and Perplexity AI, with support for multi-language content generation. It is designed for content teams or marketers who want to streamline blog planning, in-depth research, content writing, image generation, and publishing on WordPress, leveraging AI tools and Google Workspace integrations.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Configuration**: Handles triggers (schedule or form), initial settings, and retrieving input topics and categories from Google Sheets and external APIs.
- **1.2 SEO and Content Research**: Uses Langchain agents and Perplexity AI to conduct layered research on topics, chapters, and subchapters.
- **1.3 Content Generation**: Employs GPT-4o via Langchain chains to generate blog plans, chapters, and subchapters in structured formats.
- **1.4 Image Generation and Processing**: Generates AI images for featured images and chapter illustrations, processes them (resize), and uploads to Google Drive.
- **1.5 Data Aggregation and Formatting**: Aggregates chapter and tag data, compiles the full article, converts markdown content to HTML, and prepares Google Docs.
- **1.6 WordPress Publishing and Tag Management**: Manages WordPress post creation, tag existence checks/creation, setting featured images, and updating excerpts.
- **1.7 Workflow Control and Batch Processing**: Implements batching, waiting, splitting, and merging logic to handle asynchronous operations and large data sets.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Configuration

**Overview:**  
This block initializes the workflow either by scheduled trigger or form submission, sets global configurations, and retrieves initial blog topics and categories from Google Sheets and external APIs.

**Nodes Involved:**  
- Schedule Trigger  
- On form submission  
- When Executed by Another Workflow  
- Global Configuration  
- Settings  
- Google Sheets Create Topics  
- Google Sheets Create Topic - Add New Topic  
- Google Sheets In Progress Topic  
- HTTP Request Get Categories  
- Edit Fields Categories  
- Aggregate Categories  
- Check Inputs

**Node Details:**

- **Schedule Trigger**  
  - *Type:* Trigger node  
  - *Role:* Initiates workflow on a defined schedule.  
  - *Connections:* Outputs to "Google Sheets Create Topics".  
  - *Edge Cases:* Misconfigured schedule may cause unwanted runs.

- **On form submission**  
  - *Type:* Webhook trigger  
  - *Role:* Starts workflow when a form is submitted.  
  - *Connections:* Outputs to "Google Sheets Create Topic - Add New Topic".  
  - *Edge Cases:* Webhook authorization or connectivity issues.

- **When Executed by Another Workflow**  
  - *Type:* ExecuteWorkflowTrigger  
  - *Role:* Allows external workflows to trigger this one.  
  - *Connections:* Outputs to "Global Configuration".  
  - *Edge Cases:* Input parameter mismatches.

- **Global Configuration**  
  - *Type:* Set  
  - *Role:* Sets global variables or parameters used throughout the workflow.  
  - *Connections:* Outputs to "Google Sheets In Progress Topic".  
  - *Edge Cases:* Missing or incorrect configuration may propagate errors.

- **Settings**  
  - *Type:* Set  
  - *Role:* Further parameter setup or environment settings.  
  - *Connections:* Outputs to "HTTP Request Get Categories".  
  - *Edge Cases:* Conflicting parameters.

- **Google Sheets Create Topics**  
  - *Type:* Google Sheets  
  - *Role:* Retrieves or creates blog topics in a Google Sheet.  
  - *Connections:* Outputs to "Loop Over Topics".  
  - *Edge Cases:* API quota limits, sheet access permissions.

- **Google Sheets Create Topic - Add New Topic**  
  - *Type:* Google Sheets  
  - *Role:* Adds new topics from form submissions to Google Sheets.  
  - *Edge Cases:* Write permission issues.

- **Google Sheets In Progress Topic**  
  - *Type:* Google Sheets  
  - *Role:* Tracks topics currently being processed.  
  - *Connections:* Outputs to "Settings".  
  - *Edge Cases:* Sync or concurrency issues.

- **HTTP Request Get Categories**  
  - *Type:* HTTP Request  
  - *Role:* Fetches categories from external API or WordPress.  
  - *Connections:* Outputs to "Edit Fields Categories".  
  - *Edge Cases:* API authentication errors, timeout.

- **Edit Fields Categories**  
  - *Type:* Set  
  - *Role:* Normalizes or formats category data.  
  - *Connections:* Outputs to "Aggregate Categories".  
  - *Edge Cases:* Incorrect data format.

- **Aggregate Categories**  
  - *Type:* Aggregate  
  - *Role:* Consolidates category data for further use.  
  - *Connections:* Outputs to "Check Inputs".  
  - *Edge Cases:* Empty input arrays.

- **Check Inputs**  
  - *Type:* If  
  - *Role:* Verifies inputs and routes workflow to research or sitemap retrieval.  
  - *Connections:* Outputs to "Initial Research" or "Get Post Sitemap".  
  - *Edge Cases:* Missing or empty inputs.

---

#### 1.2 SEO and Content Research

**Overview:**  
Conducts multi-level AI-driven research on the blog topic, chapters, and subchapters using GPT-4o and Perplexity AI to gather content ideas and data.

**Nodes Involved:**  
- Initial Research (Langchain agent)  
- Message a model in Perplexity (Perplexity AI node)  
- Structured Output Parser4  
- Wait6  
- Get Initial Research Data  
- Split Out Main Chapters  
- Loop Over Chapters2  
- Main Chapter Researcher (Langchain agent)  
- Message a model in Perplexity1  
- Wait7  
- Chapter Copywriter  
- Structured Output Parser5  
- Loop Over Chapter  
- Final Chapters Output  
- Merge Chapters  
- Loop Over Sub-Chapter  
- Subchapter Researcher (Langchain agent)  
- Message a model in Perplexity2  
- Wait4  
- Subchapter Copywriter  
- Structured Output Parser3  
- Wait2  

**Node Details:**

- **Initial Research**  
  - *Type:* Langchain agent node  
  - *Role:* Performs initial deep research on the input topic using GPT-4o.  
  - *Retry:* Enabled with 5000ms delay between retries.  
  - *Outputs:* Feeds into "Wait6" for pacing.  
  - *Edge Cases:* API rate limits, incomplete or empty responses.

- **Message a model in Perplexity**  
  - *Type:* Perplexity AI node  
  - *Role:* Queries Perplexity AI for additional research data.  
  - *Connected to:* "Initial Research" (as AI tool integration).  
  - *Edge Cases:* API key issues, latency, or no-reply.

- **Structured Output Parser4**  
  - *Type:* Langchain structured output parser  
  - *Role:* Parses AI output into structured data.  
  - *Connected to:* "Initial Research".  
  - *Edge Cases:* Parsing errors if AI output format changes.

- **Wait6**  
  - *Type:* Wait node  
  - *Role:* Introduces delay to ensure API pacing and throttling compliance.  
  - *Outputs:* "Get Initial Research Data".

- **Get Initial Research Data**  
  - *Type:* Set  
  - *Role:* Prepares research data for downstream splitting.  
  - *Outputs:* "Split Out Main Chapters".

- **Split Out Main Chapters**  
  - *Type:* SplitOut  
  - *Role:* Splits main chapters from initial research data into individual items for processing.  
  - *Outputs:* Branches to "Merge3" and "Loop Over Chapters2".

- **Loop Over Chapters2**  
  - *Type:* SplitInBatches  
  - *Role:* Processes chapters in batches to avoid API overload.  
  - *Outputs:* "Get Title Chapter And Content1" and "Main Chapter Researcher".

- **Main Chapter Researcher**  
  - *Type:* Langchain agent  
  - *Role:* Performs deep AI research on each chapter using GPT-4o and Perplexity AI integration.  
  - *Retry:* Enabled with 5000ms delay.  
  - *Outputs:* "Wait7".

- **Message a model in Perplexity1**  
  - *Type:* Perplexity AI node  
  - *Role:* Supports "Main Chapter Researcher" with AI-generated content.  

- **Wait7**  
  - *Type:* Wait node  
  - *Role:* Throttling delay after main chapter research.  
  - *Outputs:* "Chapter Copywriter".

- **Chapter Copywriter**  
  - *Type:* Langchain chain LLM  
  - *Role:* Generates chapter content in structured form using GPT-4o.  
  - *Retry:* Enabled with 5000ms delay.  
  - *Outputs:* "Wait5".

- **Structured Output Parser5**  
  - *Type:* Output parser  
  - *Role:* Parses chapter copywriter output.  
  - *Connected to:* "Chapter Copywriter".

- **Loop Over Chapter**  
  - *Type:* SplitInBatches  
  - *Role:* Iterates chapters for further processing.  
  - *Branches:* Outputs to "Final Chapters Output" and "Edit Fields Subchapters".

- **Final Chapters Output**  
  - *Type:* Set  
  - *Role:* Prepares aggregated chapter data.  
  - *Outputs:* "Merge Chapters".

- **Merge Chapters**  
  - *Type:* Merge  
  - *Role:* Aggregates all chapter data into one dataset.  
  - *Outputs:* "Aggregate All Chapters".

- **Loop Over Sub-Chapter**  
  - *Type:* SplitInBatches  
  - *Role:* Iterates through subchapters for detailed research and writing.  
  - *Outputs:* "Loop Over Chapter" and "Subchapter Researcher".

- **Subchapter Researcher**  
  - *Type:* Langchain agent  
  - *Role:* Performs AI research on subchapters with retry and delay.  
  - *Outputs:* "Wait4".

- **Message a model in Perplexity2**  
  - *Type:* Perplexity AI node  
  - *Role:* Supports subchapter research.

- **Wait4**  
  - *Type:* Wait node  
  - *Role:* Delay after subchapter research.  
  - *Outputs:* "Subchapter Copywriter".

- **Subchapter Copywriter**  
  - *Type:* Langchain chain LLM  
  - *Role:* Generates subchapter content in structured format.  
  - *Retry:* Enabled with delay.  
  - *Outputs:* "Wait2".

- **Structured Output Parser3**  
  - *Type:* Output parser  
  - *Role:* Parses subchapter copywriter output.

- **Wait2**  
  - *Type:* Wait node  
  - *Role:* Delay to respect API limits.

---

#### 1.3 Content Generation

**Overview:**  
Generates blog plans, chapters, and subchapters using GPT-4o, parses outputs into structured data, and prepares the content for assembly.

**Nodes Involved:**  
- Blog Planner (Langchain chain LLM)  
- Structured Output Parser1  
- Check empty output (If)  
- Create Drive Folder (Google Drive)  
- Get Output (Set)  
- Split Out Chapters (SplitOut)  
- Loop Over Items (SplitInBatches)  
- Combine Into Article (Code)  
- Final Article In Markdown (Set)  
- Markdown To HTML (Markdown)  
- FInal Article In HTML (Set)  
- Create Doc (Google Docs)  
- Save Texts To Doc (Google Docs)  

**Node Details:**

- **Blog Planner**  
  - *Type:* Langchain chain LLM  
  - *Role:* Creates an overall blog plan including chapter outlines.  
  - *Retry:* Enabled with delay.  
  - *Outputs:* "Check empty output".

- **Structured Output Parser1**  
  - *Type:* Output parser for Blog Planner.

- **Check empty output**  
  - *Type:* If  
  - *Role:* Validates blog plan output; branches to "Create Drive Folder" or loops back to planner.  
  - *Retry:* Enabled.

- **Create Drive Folder**  
  - *Type:* Google Drive  
  - *Role:* Creates a folder for the blog project.  
  - *Outputs:* "Get Output".

- **Get Output**  
  - *Type:* Set  
  - *Role:* Prepares structured output data for chapter splitting and tag handling.  
  - *Outputs:* "Split Out Chapters", "Split Out Tags", "HTTP Request Open AI Image 1".

- **Split Out Chapters**  
  - *Type:* SplitOut  
  - *Role:* Divides chapters into separate processing items.  
  - *Outputs:* "Loop Over Items".

- **Loop Over Items**  
  - *Type:* SplitInBatches  
  - *Role:* Processes chapters in manageable batches.  
  - *Outputs:* "Combine Into Article" and "HTTP Request Open AI Image".

- **Combine Into Article**  
  - *Type:* Code  
  - *Role:* Combines chapter content into a single markdown article.  
  - *Outputs:* "Final Article In Markdown".

- **Final Article In Markdown**  
  - *Type:* Set  
  - *Role:* Stores combined markdown content.  
  - *Outputs:* "Markdown To HTML".

- **Markdown To HTML**  
  - *Type:* Markdown node  
  - *Role:* Converts markdown content to HTML.  
  - *Outputs:* "FInal Article In HTML".

- **FInal Article In HTML**  
  - *Type:* Set  
  - *Role:* Holds final HTML content.  
  - *Outputs:* "Create Doc".

- **Create Doc**  
  - *Type:* Google Docs  
  - *Role:* Creates a Google Doc with the blog content.  
  - *Outputs:* "Save Texts To Doc".

- **Save Texts To Doc**  
  - *Type:* Google Docs  
  - *Role:* Saves or updates the Google Doc.  
  - *Outputs:* "Merge Tags".

---

#### 1.4 Image Generation and Processing

**Overview:**  
Generates AI images for blog chapters and featured images, processes images (resizing), uploads to Google Drive, and updates metadata.

**Nodes Involved:**  
- HTTP Request Open AI Image (Chapter images)  
- HTTP Request Open AI Image 1 (Featured images)  
- Split Out Chapter Image  
- Filter Chapter Image Data  
- Convert Data To File  
- Resize Image  
- Upload Chapter Images  
- Update Image Meta Data  
- Upload Chapter Images To Drive  
- Resize Featured Image  
- Upload Featured Image To Drive  
- Merge2  
- Wait Featured Image  
- Upload Featured Image  
- Update Featured Image Meta Data  

**Node Details:**

- **HTTP Request Open AI Image**  
  - *Type:* HTTP Request  
  - *Role:* Calls OpenAI or image generation API for chapter images.  
  - *Outputs:* "Split Out Chapter Image".

- **Split Out Chapter Image**  
  - *Type:* SplitOut  
  - *Role:* Splits image data per chapter.  
  - *Outputs:* "Filter Chapter Image Data".

- **Filter Chapter Image Data**  
  - *Type:* Filter  
  - *Role:* Ensures only valid image data proceeds.  
  - *Outputs:* "Convert Data To File".

- **Convert Data To File**  
  - *Type:* ConvertToFile  
  - *Role:* Converts image data to file format.  
  - *Outputs:* "Resize Image" and "Upload Chapter Images To Drive".

- **Resize Image**  
  - *Type:* Edit Image  
  - *Role:* Resizes chapter images.  
  - *Outputs:* "Upload Chapter Images".

- **Upload Chapter Images**  
  - *Type:* HTTP Request  
  - *Role:* Uploads images to hosting or CDN.  
  - *Outputs:* "Update Image Meta Data".

- **Update Image Meta Data**  
  - *Type:* HTTP Request  
  - *Role:* Updates metadata for uploaded images.  
  - *Outputs:* "Merge1".

- **Upload Chapter Images To Drive**  
  - *Type:* Google Drive  
  - *Role:* Uploads images to Google Drive folder.  
  - *Outputs:* "Merge1".

- **HTTP Request Open AI Image 1**  
  - *Type:* HTTP Request  
  - *Role:* Generates featured images.  
  - *Outputs:* "Split Out Feature Image".

- **Split Out Feature Image**  
  - *Type:* SplitOut  
  - *Role:* Splits featured image data.  
  - *Outputs:* "Filter Feature Image Data".

- **Filter Feature Image Data**  
  - *Type:* Filter  
  - *Role:* Validates featured image data.  
  - *Outputs:* "Convert to File".

- **Convert to File**  
  - *Type:* ConvertToFile  
  - *Role:* Converts featured images to file format.  
  - *Outputs:* "Resize Featured Image" and "Upload Featured Image To Drive".

- **Resize Featured Image**  
  - *Type:* Edit Image  
  - *Role:* Resizes featured image.  
  - *Outputs:* "Merge2".

- **Upload Featured Image To Drive**  
  - *Type:* Google Drive  
  - *Role:* Uploads featured image to Drive.  
  - *Outputs:* "Merge2".

- **Merge2**  
  - *Type:* Merge  
  - *Role:* Joins resized and uploaded featured image data.  
  - *Outputs:* "Wait Featured Image".

- **Wait Featured Image**  
  - *Type:* Merge  
  - *Role:* Synchronizes image upload completion.  
  - *Outputs:* "Upload Featured Image".

- **Upload Featured Image**  
  - *Type:* HTTP Request  
  - *Role:* Uploads featured image to WordPress or CDN.  
  - *Outputs:* "Update Featured Image Meta Data".

- **Update Featured Image Meta Data**  
  - *Type:* HTTP Request  
  - *Role:* Updates metadata for the featured image.  
  - *Outputs:* "Set Featured Image For Post".

---

#### 1.5 Data Aggregation and Formatting

**Overview:**  
Aggregates tags, chapters, and content data; converts and formats content; prepares for publishing and documentation.

**Nodes Involved:**  
- Split Out Tags  
- Loop Over Tags  
- HTTP Request Get Tags  
- If Tag Already Exists  
- HTTP Request Create Tag  
- If New Tag  
- Edit Fields  
- Edit Fields2  
- Aggregate Tag IDs  
- Edit Fields Final Tags  
- Aggregate All Chapters  
- Blog Planner  
- Merge Tags  
- Post On Wordpress  
- Set Featured Image For Post  
- Set Excerpt  
- Google Sheets Final Blog  
- Google Sheets Status Update  

**Node Details:**

- **Split Out Tags**  
  - *Type:* SplitOut  
  - *Role:* Splits tags for processing.  
  - *Outputs:* "Loop Over Tags".

- **Loop Over Tags**  
  - *Type:* SplitInBatches  
  - *Role:* Processes tags in batches.  
  - *Outputs:* "Aggregate Tag IDs" and "Wait".

- **HTTP Request Get Tags**  
  - *Type:* HTTP Request  
  - *Role:* Retrieves existing tags from WordPress or API.  
  - *OnError:* Continues on error.  
  - *Outputs:* "If Tag Already Exists" or "Loop Over Tags".

- **If Tag Already Exists**  
  - *Type:* If  
  - *Role:* Checks if a tag exists; routes accordingly.  
  - *Outputs:* "Edit Fields" for existing tags, or "HTTP Request Create Tag" for new tags.

- **HTTP Request Create Tag**  
  - *Type:* HTTP Request  
  - *Role:* Creates new tags if missing.  
  - *OnError:* Continues on error.  
  - *Outputs:* "If New Tag".

- **If New Tag**  
  - *Type:* If  
  - *Role:* Confirms tag creation success; routes accordingly.  
  - *Outputs:* "Edit Fields2" or re-loop on tags.

- **Edit Fields / Edit Fields2**  
  - *Type:* Set  
  - *Role:* Prepares tag data for aggregation or looping.

- **Aggregate Tag IDs**  
  - *Type:* Aggregate  
  - *Role:* Gathers tag IDs for post tagging.  
  - *Outputs:* "Edit Fields Final Tags".

- **Edit Fields Final Tags**  
  - *Type:* Set  
  - *Role:* Finalizes tag data before merging.  
  - *Outputs:* "Merge Tags".

- **Aggregate All Chapters**  
  - *Type:* Aggregate  
  - *Role:* Collects all chapter content for the blog.  
  - *Outputs:* "Blog Planner".

- **Merge Tags**  
  - *Type:* Merge  
  - *Role:* Combines tag data for final post creation.  
  - *Outputs:* "Post On Wordpress".

- **Post On Wordpress**  
  - *Type:* WordPress  
  - *Role:* Creates or updates the WordPress post.  
  - *Outputs:* "Wait Featured Image".

- **Set Featured Image For Post**  
  - *Type:* HTTP Request  
  - *Role:* Assigns featured image to the WordPress post.  
  - *Outputs:* "Set Excerpt".

- **Set Excerpt**  
  - *Type:* HTTP Request  
  - *Role:* Sets post excerpt on WordPress.  
  - *Outputs:* "Google Sheets Final Blog".

- **Google Sheets Final Blog**  
  - *Type:* Google Sheets  
  - *Role:* Updates final blog status or metadata in Google Sheets.  
  - *Outputs:* "Google Sheets Status Update".

- **Google Sheets Status Update**  
  - *Type:* Google Sheets  
  - *Role:* Final status update after publishing.

---

#### 1.6 Workflow Control and Batch Processing

**Overview:**  
Manages splitting, batching, waiting, merging, and controlling execution flow to handle asynchronous AI calls, API rate limits, and large data sets.

**Nodes Involved:**  
- SplitOut nodes (Chapters, Subchapters, Links, Tags, Images)  
- SplitInBatches nodes (Loop Over Chapters, Subchapters, Tags, Items, Topics)  
- Merge nodes (Merge1, Merge2, Merge3, Merge Chapters, Merge Tags)  
- Wait nodes (Wait, Wait1 to Wait7)  
- Switch Deep Research  
- If nodes (Check Inputs, Check empty output, If Tag Already Exists, If New Tag)  
- Aggregate nodes (Aggregate, Aggregate Categories, Aggregate Internal Links, Aggregate Tag IDs, Aggregate All Chapters)

**Node Details:**  
- These nodes ensure the workflow handles data in manageable chunks, waits to comply with rate limits, and merges results properly to maintain data integrity.  
- Potential failure modes include timeouts, batch size misconfiguration, and race conditions.

---

### 3. Summary Table

| Node Name                         | Node Type               | Functional Role                          | Input Node(s)                           | Output Node(s)                        | Sticky Note |
|----------------------------------|-------------------------|----------------------------------------|---------------------------------------|-------------------------------------|-------------|
| Schedule Trigger                 | Trigger                 | Starts workflow on schedule             |                                       | Google Sheets Create Topics          |             |
| On form submission              | Webhook Trigger         | Starts workflow on form submission      |                                       | Google Sheets Create Topic - Add New Topic |             |
| When Executed by Another Workflow | ExecuteWorkflowTrigger  | Triggered by external workflow          |                                       | Global Configuration                 |             |
| Global Configuration            | Set                     | Sets global parameters                   | When Executed by Another Workflow     | Google Sheets In Progress Topic      |             |
| Google Sheets In Progress Topic | Google Sheets            | Tracks in-progress topics                | Global Configuration                  | Settings                            |             |
| Settings                       | Set                     | Environment and parameter setup          | Google Sheets In Progress Topic       | HTTP Request Get Categories          |             |
| HTTP Request Get Categories     | HTTP Request            | Retrieves category data                   | Settings                            | Edit Fields Categories               |             |
| Edit Fields Categories          | Set                     | Formats category data                     | HTTP Request Get Categories            | Aggregate Categories                 |             |
| Aggregate Categories            | Aggregate               | Aggregates categories                     | Edit Fields Categories                 | Check Inputs                       |             |
| Check Inputs                   | If                      | Validates inputs and directs flow        | Aggregate Categories                   | Initial Research, Get Post Sitemap  |             |
| Initial Research              | Langchain Agent          | Performs initial deep research            | Check Inputs                        | Wait6                              |             |
| Message a model in Perplexity   | Perplexity AI           | Supports AI research                      | Initial Research                    | Initial Research                    |             |
| Structured Output Parser4       | Output Parser           | Parses initial research output            | Initial Research                    |                                   |             |
| Wait6                         | Wait                    | Introduces delay after initial research   | Initial Research                    | Get Initial Research Data           |             |
| Get Initial Research Data       | Set                     | Prepares research data                     | Wait6                             | Split Out Main Chapters             |             |
| Split Out Main Chapters         | SplitOut                | Splits chapters from research data        | Get Initial Research Data           | Merge3, Loop Over Chapters2         |             |
| Merge3                        | Merge                   | Merges main chapters and internal links   | Split Out Main Chapters, Aggregate Internal Links | Switch Deep Research               |             |
| Aggregate Internal Links        | Aggregate               | Aggregates internal links                   | Limit Internal Links                | Merge3                            |             |
| Limit Internal Links            | Limit                   | Limits internal links count                 | Split Out Links                   | Aggregate Internal Links            |             |
| Split Out Links                 | SplitOut                | Splits sitemap links                        | Get XML File                     | Limit Internal Links                |             |
| Get XML File                   | XML                     | Parses XML sitemap                          | Get Post Sitemap                 | Split Out Links                   |             |
| Get Post Sitemap               | HTTP Request            | Retrieves sitemap data                       | Check Inputs                    | Get XML File                    |             |
| Loop Over Chapters2             | SplitInBatches          | Processes chapters in batches                | Split Out Main Chapters           | Get Title Chapter And Content1, Main Chapter Researcher |             |
| Get Title Chapter And Content1  | Set                     | Prepares chapter title and content          | Loop Over Chapters2              | Merge Chapters                   |             |
| Main Chapter Researcher         | Langchain Agent         | Performs deep research on chapters           | Loop Over Chapters2              | Wait7                          |             |
| Message a model in Perplexity1  | Perplexity AI           | Supports main chapter research              | Main Chapter Researcher          | Main Chapter Researcher           |             |
| Wait7                         | Wait                    | Delay after main chapter research            | Main Chapter Researcher          | Chapter Copywriter               |             |
| Chapter Copywriter              | Langchain Chain LLM     | Generates chapter content                     | Wait7                          | Wait5                          |             |
| Structured Output Parser5       | Output Parser           | Parses chapter content output                 | Chapter Copywriter              |                               |             |
| Wait5                         | Wait                    | Delay after chapter copywriting                | Chapter Copywriter              | Loop Over Chapters2             |             |
| Loop Over Chapter              | SplitInBatches          | Processes each chapter                          | Loop Over Sub-Chapter           | Final Chapters Output, Edit Fields Subchapters |             |
| Final Chapters Output           | Set                     | Prepares final chapters output                   | Loop Over Chapter              | Merge Chapters                 |             |
| Merge Chapters                 | Merge                   | Merges all chapter content                       | Final Chapters Output, Get Title Chapter And Content1 | Aggregate All Chapters           |             |
| Aggregate All Chapters          | Aggregate               | Aggregates all chapters data                      | Merge Chapters                 | Blog Planner                   |             |
| Blog Planner                  | Langchain Chain LLM     | Creates blog plan and outline                      | Aggregate All Chapters         | Check empty output             |             |
| Check empty output            | If                      | Validates blog plan output                          | Blog Planner                 | Create Drive Folder, Blog Planner |             |
| Create Drive Folder           | Google Drive            | Creates Drive folder for blog assets                 | Check empty output            | Get Output                    |             |
| Get Output                    | Set                     | Prepares data for chapter and tag splitting          | Create Drive Folder           | Split Out Chapters, Split Out Tags, HTTP Request Open AI Image 1 |             |
| Split Out Chapters           | SplitOut                | Splits chapters for processing                       | Get Output                   | Loop Over Items               |             |
| Loop Over Items              | SplitInBatches          | Processes chapters in batches                         | Split Out Chapters            | Combine Into Article, HTTP Request Open AI Image |             |
| Combine Into Article          | Code                    | Combines chapter content into markdown article         | Loop Over Items              | Final Article In Markdown     |             |
| Final Article In Markdown      | Set                     | Stores combined markdown content                        | Combine Into Article          | Markdown To HTML             |             |
| Markdown To HTML             | Markdown                | Converts markdown to HTML                                 | Final Article In Markdown      | FInal Article In HTML         |             |
| FInal Article In HTML          | Set                     | Holds final HTML content                                  | Markdown To HTML             | Create Doc                  |             |
| Create Doc                   | Google Docs             | Creates Google Docs document                             | FInal Article In HTML          | Save Texts To Doc            |             |
| Save Texts To Doc            | Google Docs             | Saves or updates Google Docs                              | Create Doc                  | Merge Tags                  |             |
| Split Out Tags              | SplitOut                | Splits tags for processing                                | Get Output                   | Loop Over Tags              |             |
| Loop Over Tags              | SplitInBatches          | Processes tags in batches                                  | Split Out Tags              | Aggregate Tag IDs, Wait       |             |
| Aggregate Tag IDs           | Aggregate               | Aggregates tag IDs                                         | Loop Over Tags              | Edit Fields Final Tags       |             |
| Edit Fields Final Tags       | Set                     | Prepares final tag data                                    | Aggregate Tag IDs           | Merge Tags                  |             |
| Merge Tags                  | Merge                   | Merges tag data for final post                             | Edit Fields Final Tags       | Post On Wordpress           |             |
| HTTP Request Get Tags        | HTTP Request            | Retrieves existing tags                                    | Wait                       | If Tag Already Exists, Loop Over Tags |             |
| If Tag Already Exists        | If                      | Checks for tag existence                                   | HTTP Request Get Tags         | Edit Fields, HTTP Request Create Tag |             |
| HTTP Request Create Tag      | HTTP Request            | Creates new tags                                          | If Tag Already Exists        | If New Tag                 |             |
| If New Tag                  | If                      | Confirms new tag creation                                  | HTTP Request Create Tag       | Edit Fields2, Loop Over Tags |             |
| Edit Fields                  | Set                     | Prepares existing tag data                                 | If Tag Already Exists        | Loop Over Tags              |             |
| Edit Fields2                 | Set                     | Prepares new tag data                                      | If New Tag                  | Loop Over Tags              |             |
| Post On Wordpress            | WordPress               | Publishes blog post                                       | Merge Tags                  | Wait Featured Image         |             |
| Wait Featured Image          | Merge                   | Synchronizes featured image upload                         | Post On Wordpress            | Upload Featured Image       |             |
| Upload Featured Image        | HTTP Request            | Uploads featured image                                    | Wait Featured Image          | Update Featured Image Meta Data |             |
| Update Featured Image Meta Data | HTTP Request          | Updates metadata for featured image                         | Upload Featured Image        | Set Featured Image For Post |             |
| Set Featured Image For Post  | HTTP Request            | Sets featured image on WordPress post                      | Update Featured Image Meta Data | Set Excerpt                |             |
| Set Excerpt                 | HTTP Request            | Sets post excerpt                                        | Set Featured Image For Post  | Google Sheets Final Blog    |             |
| Google Sheets Final Blog     | Google Sheets           | Updates blog status and metadata                            | Set Excerpt                 | Google Sheets Status Update |             |
| Google Sheets Status Update | Google Sheets           | Final status update                                       | Google Sheets Final Blog     |                             |             |
| HTTP Request Open AI Image  | HTTP Request            | Generates chapter images                                   | Loop Over Items             | Split Out Chapter Image     |             |
| Split Out Chapter Image      | SplitOut                | Splits chapter images                                      | HTTP Request Open AI Image  | Filter Chapter Image Data   |             |
| Filter Chapter Image Data    | Filter                  | Filters valid chapter image data                           | Split Out Chapter Image      | Convert Data To File        |             |
| Convert Data To File         | ConvertToFile           | Converts image data to files                               | Filter Chapter Image Data    | Resize Image, Upload Chapter Images To Drive |             |
| Resize Image                | Edit Image              | Resizes chapter images                                    | Convert Data To File         | Upload Chapter Images       |             |
| Upload Chapter Images        | HTTP Request            | Uploads chapter images                                    | Resize Image                | Update Image Meta Data      |             |
| Update Image Meta Data       | HTTP Request            | Updates metadata for chapter images                         | Upload Chapter Images        | Merge1                     |             |
| Upload Chapter Images To Drive | Google Drive           | Uploads chapter images to Drive                             | Convert Data To File         | Merge1                     |             |
| Merge1                      | Merge                   | Merges chapter image upload data                           | Update Image Meta Data, Upload Chapter Images To Drive | Merge1 outputs to Merge1 or Merge3 |             |
| HTTP Request Open AI Image 1 | HTTP Request            | Generates featured images                                  | Get Output                  | Split Out Feature Image    |             |
| Split Out Feature Image      | SplitOut                | Splits featured images                                    | HTTP Request Open AI Image 1 | Filter Feature Image Data  |             |
| Filter Feature Image Data    | Filter                  | Filters valid featured image data                          | Split Out Feature Image      | Convert to File            |             |
| Convert to File             | ConvertToFile           | Converts featured image data to files                      | Filter Feature Image Data    | Resize Featured Image, Upload Featured Image To Drive |             |
| Resize Featured Image        | Edit Image              | Resizes featured image                                    | Convert to File             | Merge2                     |             |
| Upload Featured Image To Drive | Google Drive           | Uploads featured image to Drive                             | Convert to File             | Merge2                     |             |
| Merge2                      | Merge                   | Merges featured image upload data                          | Resize Featured Image, Upload Featured Image To Drive | Wait Featured Image         |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**  
   - Add a *Schedule Trigger* for periodic execution.  
   - Add a *Webhook Trigger* node named "On form submission" for manual topic input.  
   - Add an *Execute Workflow Trigger* node for external workflow invocations.

2. **Set Global Parameters:**  
   - Add a *Set* node "Global Configuration" to define key workflow parameters, e.g., API keys, default language, batch sizes.

3. **Topic and Category Input:**  
   - Add *Google Sheets* nodes:  
     - "Google Sheets Create Topics" to read planned topics.  
     - "Google Sheets Create Topic - Add New Topic" to add new topics from form webhook.  
     - "Google Sheets In Progress Topic" to track ongoing topics.  
   - Add an *HTTP Request* node "HTTP Request Get Categories" to fetch categories from WordPress or external API.  
   - Add *Set* and *Aggregate* nodes to process and consolidate category data.

4. **Input Validation:**  
   - Add an *If* node "Check Inputs" to verify that required inputs (topics, categories) are present before continuing.

5. **Research Layer:**  
   - Create Langchain agent nodes:  
     - "Initial Research" for broad topic research.  
     - "Main Chapter Researcher" for detailed chapter research.  
     - "Subchapter Researcher" for subchapter-level research.  
   - Integrate Perplexity AI nodes to support research agents.  
   - Add structured output parsers for AI outputs.  
   - Insert *Wait* nodes to manage pacing and API rate limits.  
   - Use *SplitOut* and *SplitInBatches* nodes to handle main chapters and subchapters processing in batches.

6. **Content Generation:**  
   - Add Langchain chain LLM nodes:  
     - "Blog Planner" to create blog outline and chapter titles.  
     - "Chapter Copywriter" and "Subchapter Copywriter" to generate content.  
   - Add structured output parsers for these content nodes.  
   - Manage batch processing and splitting of chapters and subchapters.

7. **Content Assembly:**  
   - Use *Set* nodes to prepare data for merging.  
   - Add *Merge* nodes to combine chapters and subchapters.  
   - Use a *Code* node "Combine Into Article" for merging markdown content.  
   - Add *Markdown* node to convert markdown to HTML.  
   - Add *Google Docs* nodes to create and save the blog document.

8. **Image Generation and Processing:**  
   - Add *HTTP Request* nodes to call OpenAI or similar for chapter and featured image generation.  
   - Use *SplitOut* and *Filter* nodes to process image data.  
   - Use *ConvertToFile* and *Edit Image* nodes to convert and resize images.  
   - Upload images to Google Drive using *Google Drive* nodes.  
   - Add HTTP Request nodes to upload images to WordPress and update metadata.

9. **Tag Management:**  
   - Use *SplitOut* and *SplitInBatches* nodes to process tags.  
   - Use *HTTP Request* nodes to get existing tags and create new tags as needed.  
   - Use *If* nodes to check tag existence and creation.  
   - Use *Set* and *Aggregate* nodes to prepare tags for post assignment.

10. **WordPress Publishing:**  
    - Add a *WordPress* node to create or update the blog post.  
    - Add HTTP Request nodes to assign the featured image and set the post excerpt.  
    - Add *Google Sheets* nodes to update the blog status and metadata.

11. **Flow Control:**  
    - Add *Wait* nodes to handle API rate limits and flow pacing.  
    - Use *Merge* nodes to synchronize parallel branches.  
    - Use *Switch* and *If* nodes to control branching logic, such as research depth.

12. **Test and Validate:**  
    - Test the workflow in stages, verify data aggregation, AI output formatting, image handling, and WordPress publishing.  
    - Handle potential edge cases like API failures, empty content, or authorization errors with retry and error handling strategies.

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                                                                      |
|---------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| The workflow is tagged as "Template" and designed for marketplace distribution.                                | Workflow metadata                                                                                   |
| Uses advanced AI integrations including GPT-4o (OpenAI) and Perplexity AI for deep content research.           | Node types: Langchain agent, Perplexity Tool                                                      |
| Integrates extensively with Google Workspace (Sheets, Docs, Drive) for content management and storage.         | Google Sheets nodes for input and status tracking, Google Docs for content output, Drive for images |
| WordPress node and HTTP Request nodes handle publishing and media uploads on WordPress sites.                  | WordPress publishing nodes and HTTP requests for images and metadata                              |
| Includes batching and waiting nodes to comply with API rate limits and ensure stable asynchronous processing.  | SplitInBatches and Wait nodes throughout the workflow                                              |
| Sticky Notes are present but empty; recommended to add documentation comments in the workflow editor.          | Multiple "Sticky Note" nodes exist but lack content                                               |
| For more details on Langchain nodes and Perplexity AI nodes, refer to n8n official documentation and forums.  | n8n docs: https://docs.n8n.io/nodes/                                                               |

---

**Disclaimer:** The provided documentation is based solely on the analyzed n8n workflow JSON. The workflow respects all applicable content policies and handles only legal and public data.
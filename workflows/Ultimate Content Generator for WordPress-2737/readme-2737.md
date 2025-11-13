Ultimate Content Generator for WordPress

https://n8nworkflows.xyz/workflows/ultimate-content-generator-for-wordpress-2737


# Ultimate Content Generator for WordPress

### 1. Workflow Overview

The **Ultimate Content Generator for WordPress** workflow automates the entire content creation lifecycle for WordPress blogs, integrating AI content generation, SEO optimization, content management, image creation, and publishing. It is designed for content creators, marketers, and business owners who want to streamline and scale their content production with minimal manual effort.

The workflow is logically divided into the following blocks:

- **1.1 Trigger & Input Reception:** Manual or scheduled start, receiving content ideas and brand guidelines from Airtable.
- **1.2 Content Generation:** AI-driven creation of blog post titles, structure, and chapter texts tailored to brand tone and SEO.
- **1.3 Content Validation & Assembly:** Data consistency checks, splitting and merging chapter content into a final article.
- **1.4 SEO Optimization:** Integration with RankMath SEO plugin to update focus keywords and meta descriptions.
- **1.5 Featured Image Creation:** AI-powered generation of branded featured images, uploading and associating them with posts.
- **1.6 Publishing & Status Update:** Posting the content to WordPress, updating Airtable status, and notifying via Slack.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Input Reception

**Overview:**  
This block initiates the workflow either manually or via Airtable triggers, then retrieves brand guidelines and keywords from Airtable to prepare for content generation.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- Airtable Trigger  
- GET Brand Guidelines (Airtable)  
- Aggregate (Aggregate)  
- GET Keywords (Airtable)  
- If (Conditional Check)  
- Post Parameters (Set)  
- Loop Over Items (Split in Batches)  
- Settings (Set)

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Allows manual start for testing or immediate execution.  
  - Connections: Outputs to GET Brand Guidelines.  
  - Edge Cases: No input data; manual trigger only.

- **Airtable Trigger**  
  - Type: Airtable Trigger  
  - Role: Automatically triggers workflow on Airtable record changes.  
  - Connections: Outputs to GET Brand Guidelines.  
  - Edge Cases: Airtable API downtime or auth errors.

- **GET Brand Guidelines**  
  - Type: Airtable  
  - Role: Fetches brand tone, voice, and style guidelines.  
  - Configuration: Airtable base and table configured for brand data.  
  - Connections: Outputs to Aggregate node.  
  - Edge Cases: Missing or malformed brand data.

- **Aggregate**  
  - Type: Aggregate  
  - Role: Consolidates brand guideline data into a single object.  
  - Connections: Outputs to GET Keywords.  
  - Edge Cases: Empty aggregation if no brand data.

- **GET Keywords**  
  - Type: Airtable  
  - Role: Retrieves SEO keywords for the post.  
  - Connections: Outputs to If node.  
  - Edge Cases: No keywords found.

- **If**  
  - Type: Conditional  
  - Role: Checks if keywords exist to continue or halt.  
  - Connections: True branch to Post Parameters; False branch empty.  
  - Edge Cases: Expression errors if data missing.

- **Post Parameters**  
  - Type: Set  
  - Role: Prepares parameters for content generation (e.g., title, keywords).  
  - Connections: Outputs to Loop Over Items.  
  - Edge Cases: Missing parameters.

- **Loop Over Items**  
  - Type: Split In Batches  
  - Role: Processes content items in batches for scalability.  
  - Connections: True branch to Settings node.  
  - Edge Cases: Batch size misconfiguration.

- **Settings**  
  - Type: Set  
  - Role: Sets AI prompt settings and other parameters for generation.  
  - Connections: Outputs to Create post title and structure node (next block).  
  - Edge Cases: Missing or invalid prompt settings.

---

#### 1.2 Content Generation

**Overview:**  
Generates the blog post title, structure, and chapter texts using OpenAI GPT models, incorporating brand voice and SEO keywords.

**Nodes Involved:**  
- Create post title and structure (OpenAI)  
- Check data consistency (If)  
- Split out chapters (Split Out)  
- Create chapters text (OpenAI)  
- Merge chapters title and text (Merge)  
- Final article text (Code)

**Node Details:**

- **Create post title and structure**  
  - Type: OpenAI GPT (Langchain)  
  - Role: Generates SEO-friendly blog post title and outline with headings.  
  - Configuration: Custom prompt including brand guidelines and keywords.  
  - Connections: Outputs to Check data consistency.  
  - Edge Cases: API rate limits, prompt failures.

- **Check data consistency**  
  - Type: If  
  - Role: Validates AI output structure before proceeding.  
  - Connections: True to Split out chapters; False to Respond: Error.  
  - Edge Cases: Unexpected AI output format.

- **Split out chapters**  
  - Type: Split Out  
  - Role: Splits the generated outline into individual chapters for detailed content generation.  
  - Connections: Outputs to Merge chapters title and text and Create chapters text.  
  - Edge Cases: Empty or malformed chapter data.

- **Create chapters text**  
  - Type: OpenAI GPT (Langchain)  
  - Role: Generates detailed text for each chapter based on the outline.  
  - Connections: Outputs to Merge chapters title and text.  
  - Edge Cases: API failures, incomplete chapter text.

- **Merge chapters title and text**  
  - Type: Merge  
  - Role: Combines chapter titles and texts into a cohesive article structure.  
  - Connections: Outputs to Final article text.  
  - Edge Cases: Mismatched data arrays.

- **Final article text**  
  - Type: Code  
  - Role: Final formatting and cleanup of the article text before publishing.  
  - Connections: Outputs to Post on Wordpress.  
  - Edge Cases: Script errors or unexpected data.

---

#### 1.3 SEO Optimization

**Overview:**  
Updates RankMath SEO plugin with focus keywords and meta descriptions to optimize the post for search engines.

**Nodes Involved:**  
- Post on Wordpress (WordPress node)  
- SEO Update for RankMath (OpenAI)  
- POST RankMath Info (HTTP Request)

**Node Details:**

- **Post on Wordpress**  
  - Type: WordPress node  
  - Role: Publishes the generated post content to WordPress via REST API.  
  - Configuration: WordPress credentials, post parameters including content and title.  
  - Connections: Outputs to SEO Update for RankMath.  
  - Edge Cases: Authentication errors, API downtime.

- **SEO Update for RankMath**  
  - Type: OpenAI GPT (Langchain)  
  - Role: Generates SEO meta description and focus keywords for RankMath.  
  - Connections: Outputs to POST RankMath Info.  
  - Edge Cases: API failures.

- **POST RankMath Info**  
  - Type: HTTP Request  
  - Role: Sends SEO data to RankMath plugin via WordPress REST API or custom endpoint.  
  - Configuration: Auth headers, endpoint URL.  
  - Connections: Outputs to P1 Image Prompt (image generation).  
  - Edge Cases: HTTP errors, plugin API changes.

---

#### 1.4 Featured Image Creation

**Overview:**  
Generates branded featured images using AI, uploads them to WordPress media library, and associates the image with the post.

**Nodes Involved:**  
- P1 Image Prompt (OpenAI)  
- Aggregate Prompt (Aggregate)  
- Prompt Settings (Set)  
- Leo - Improve Prompt (HTTP Request)  
- Leo - Generate Image (HTTP Request)  
- Wait (Wait)  
- Leo - Get imageId (HTTP Request)  
- Download Image (HTTP Request)  
- Upload media (HTTP Request)  
- Set image ID for the post (HTTP Request)

**Node Details:**

- **P1 Image Prompt**  
  - Type: OpenAI GPT (Langchain)  
  - Role: Creates an AI prompt for image generation based on post content and brand style.  
  - Connections: Outputs to Aggregate Prompt.  
  - Edge Cases: Prompt generation errors.

- **Aggregate Prompt**  
  - Type: Aggregate  
  - Role: Combines multiple prompt elements into a single image generation prompt.  
  - Connections: Outputs to Prompt Settings.  
  - Edge Cases: Empty aggregation.

- **Prompt Settings**  
  - Type: Set  
  - Role: Sets parameters for image generation API calls.  
  - Connections: Outputs to Leo - Improve Prompt.  
  - Edge Cases: Missing parameters.

- **Leo - Improve Prompt**  
  - Type: HTTP Request  
  - Role: Sends prompt to an external service (Leo) to enhance image generation prompt.  
  - Connections: Outputs to Leo - Generate Image.  
  - Edge Cases: HTTP errors, service downtime.

- **Leo - Generate Image**  
  - Type: HTTP Request  
  - Role: Requests image generation from Leo API.  
  - Connections: Outputs to Wait.  
  - Edge Cases: API rate limits, generation failures.

- **Wait**  
  - Type: Wait  
  - Role: Delays workflow to allow image generation completion.  
  - Connections: Outputs to Leo - Get imageId.  
  - Edge Cases: Timeout if image not ready.

- **Leo - Get imageId**  
  - Type: HTTP Request  
  - Role: Retrieves generated image ID from Leo API.  
  - Connections: Outputs to Download Image.  
  - Edge Cases: Missing image ID.

- **Download Image**  
  - Type: HTTP Request  
  - Role: Downloads the generated image file.  
  - Connections: Outputs to Upload media.  
  - Edge Cases: Download failures.

- **Upload media**  
  - Type: HTTP Request  
  - Role: Uploads image to WordPress media library.  
  - Connections: Outputs to Set image ID for the post.  
  - Edge Cases: Upload errors, file size limits.

- **Set image ID for the post**  
  - Type: HTTP Request  
  - Role: Associates uploaded image ID with the WordPress post as featured image.  
  - Connections: Outputs to POST Blog Info.  
  - Edge Cases: API errors, incorrect post ID.

---

#### 1.5 Publishing & Status Update

**Overview:**  
Finalizes the publishing process by updating Airtable with post status and sending notifications via Slack.

**Nodes Involved:**  
- POST Blog Info (Slack)  
- UPDATE Status (Airtable)

**Node Details:**

- **POST Blog Info**  
  - Type: Slack  
  - Role: Sends a notification to Slack channel about the new published post.  
  - Configuration: Slack webhook or OAuth credentials.  
  - Connections: Outputs to UPDATE Status.  
  - Edge Cases: Slack API errors.

- **UPDATE Status**  
  - Type: Airtable  
  - Role: Updates the status of the content item in Airtable to reflect publishing completion.  
  - Connections: Outputs to Loop Over Items (for batch processing).  
  - Edge Cases: Airtable API errors, concurrency issues.

---

### 3. Summary Table

| Node Name                 | Node Type                      | Functional Role                          | Input Node(s)                      | Output Node(s)                   | Sticky Note                                                                                  |
|---------------------------|--------------------------------|----------------------------------------|----------------------------------|---------------------------------|----------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                 | Manual start trigger                    |                                  | GET Brand Guidelines             |                                                                                              |
| Airtable Trigger          | Airtable Trigger               | Automatic trigger on Airtable changes  |                                  | GET Brand Guidelines             |                                                                                              |
| GET Brand Guidelines      | Airtable                      | Fetch brand guidelines                  | When clicking ‘Test workflow’, Airtable Trigger | Aggregate                      |                                                                                              |
| Aggregate                 | Aggregate                     | Aggregate brand data                    | GET Brand Guidelines             | GET Keywords                    |                                                                                              |
| GET Keywords              | Airtable                      | Fetch SEO keywords                      | Aggregate                       | If                             |                                                                                              |
| If                       | Conditional                   | Check if keywords exist                 | GET Keywords                   | Post Parameters (true)           |                                                                                              |
| Post Parameters           | Set                          | Prepare parameters for generation      | If                             | Loop Over Items                 |                                                                                              |
| Loop Over Items           | Split In Batches              | Process content items in batches       | Post Parameters                 | Settings (true)                 |                                                                                              |
| Settings                  | Set                          | Set AI prompt and generation settings  | Loop Over Items                 | Create post title and structure |                                                                                              |
| Create post title and structure | OpenAI GPT (Langchain)       | Generate post title and outline        | Settings                       | Check data consistency          |                                                                                              |
| Check data consistency    | If                           | Validate AI output                      | Create post title and structure | Split out chapters (true), Respond: Error (false) |                                                                                              |
| Split out chapters        | Split Out                    | Split outline into chapters             | Check data consistency          | Merge chapters title and text, Create chapters text |                                                                                              |
| Create chapters text      | OpenAI GPT (Langchain)       | Generate detailed chapter texts         | Split out chapters              | Merge chapters title and text   |                                                                                              |
| Merge chapters title and text | Merge                        | Combine chapter titles and texts        | Split out chapters, Create chapters text | Final article text             |                                                                                              |
| Final article text        | Code                         | Final formatting of article             | Merge chapters title and text   | Post on Wordpress              |                                                                                              |
| Post on Wordpress         | WordPress                    | Publish post to WordPress               | Final article text              | SEO Update for RankMath         |                                                                                              |
| SEO Update for RankMath   | OpenAI GPT (Langchain)       | Generate SEO meta and keywords          | Post on Wordpress              | POST RankMath Info              |                                                                                              |
| POST RankMath Info        | HTTP Request                 | Send SEO data to RankMath plugin        | SEO Update for RankMath         | P1 Image Prompt                 |                                                                                              |
| P1 Image Prompt           | OpenAI GPT (Langchain)       | Generate image prompt                    | POST RankMath Info              | Aggregate Prompt               |                                                                                              |
| Aggregate Prompt          | Aggregate                    | Aggregate image prompt elements          | P1 Image Prompt                | Prompt Settings                |                                                                                              |
| Prompt Settings           | Set                          | Set parameters for image generation      | Aggregate Prompt               | Leo - Improve Prompt           |                                                                                              |
| Leo - Improve Prompt      | HTTP Request                 | Enhance prompt via external service      | Prompt Settings                | Leo - Generate Image           |                                                                                              |
| Leo - Generate Image      | HTTP Request                 | Request AI image generation              | Leo - Improve Prompt           | Wait                          |                                                                                              |
| Wait                     | Wait                         | Wait for image generation completion     | Leo - Generate Image           | Leo - Get imageId              |                                                                                              |
| Leo - Get imageId         | HTTP Request                 | Retrieve generated image ID               | Wait                          | Download Image                |                                                                                              |
| Download Image            | HTTP Request                 | Download generated image file             | Leo - Get imageId              | Upload media                  |                                                                                              |
| Upload media              | HTTP Request                 | Upload image to WordPress media library   | Download Image                | Set image ID for the post      |                                                                                              |
| Set image ID for the post | HTTP Request                 | Associate image with WordPress post        | Upload media                  | POST Blog Info                |                                                                                              |
| POST Blog Info            | Slack                        | Notify Slack about published post          | Set image ID for the post      | UPDATE Status                 |                                                                                              |
| UPDATE Status             | Airtable                     | Update content status in Airtable          | POST Blog Info                | Loop Over Items               |                                                                                              |
| Respond: Error            | Respond to Webhook           | Return error response if data invalid      | Check data consistency (false) |                               |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**  
   - Add a **Manual Trigger** node named "When clicking ‘Test workflow’" for manual starts.  
   - Add an **Airtable Trigger** node named "Airtable Trigger" configured to listen to your Airtable base and table changes.

2. **Retrieve Brand Guidelines:**  
   - Add an **Airtable** node "GET Brand Guidelines" configured to fetch brand tone, voice, and style data from your Airtable base. Connect both triggers to this node.

3. **Aggregate Brand Data:**  
   - Add an **Aggregate** node "Aggregate" to consolidate brand guidelines into a single object.

4. **Retrieve SEO Keywords:**  
   - Add an **Airtable** node "GET Keywords" to fetch SEO keywords related to the content.

5. **Conditional Check for Keywords:**  
   - Add an **If** node "If" to check if keywords exist (e.g., check if the keywords field is not empty).  
   - Connect the true branch to the next step; leave false branch empty or handle errors.

6. **Set Post Parameters:**  
   - Add a **Set** node "Post Parameters" to prepare parameters such as title, keywords, and other metadata for content generation.

7. **Batch Processing Setup:**  
   - Add a **Split In Batches** node "Loop Over Items" to process multiple content items in batches for scalability.

8. **Set AI Prompt Settings:**  
   - Add a **Set** node "Settings" to define AI prompt parameters (model, temperature, max tokens, etc.).

9. **Generate Post Title and Structure:**  
   - Add an **OpenAI GPT (Langchain)** node "Create post title and structure" with a prompt that uses brand guidelines and keywords to generate a blog post title and outline.

10. **Validate AI Output:**  
    - Add an **If** node "Check data consistency" to verify the AI output format.  
    - True branch proceeds; false branch connects to a **Respond to Webhook** node "Respond: Error" to handle errors.

11. **Split Outline into Chapters:**  
    - Add a **Split Out** node "Split out chapters" to separate the outline into individual chapters.

12. **Generate Chapter Texts:**  
    - Add an **OpenAI GPT (Langchain)** node "Create chapters text" to generate detailed content for each chapter.

13. **Merge Chapter Titles and Texts:**  
    - Add a **Merge** node "Merge chapters title and text" to combine chapter titles and texts.

14. **Finalize Article Text:**  
    - Add a **Code** node "Final article text" to format and clean up the combined article content.

15. **Publish Post on WordPress:**  
    - Add a **WordPress** node "Post on Wordpress" configured with WordPress REST API credentials to publish the post.

16. **Generate SEO Meta and Keywords:**  
    - Add an **OpenAI GPT (Langchain)** node "SEO Update for RankMath" to create SEO meta descriptions and focus keywords.

17. **Send SEO Data to RankMath:**  
    - Add an **HTTP Request** node "POST RankMath Info" to update RankMath SEO plugin via WordPress API.

18. **Generate Featured Image Prompt:**  
    - Add an **OpenAI GPT (Langchain)** node "P1 Image Prompt" to create an image generation prompt based on the post.

19. **Aggregate Image Prompt:**  
    - Add an **Aggregate** node "Aggregate Prompt" to combine prompt elements.

20. **Set Image Generation Parameters:**  
    - Add a **Set** node "Prompt Settings" to configure parameters for image generation API.

21. **Improve Image Prompt via External Service:**  
    - Add an **HTTP Request** node "Leo - Improve Prompt" to enhance the prompt.

22. **Request Image Generation:**  
    - Add an **HTTP Request** node "Leo - Generate Image" to request image generation.

23. **Wait for Image Generation:**  
    - Add a **Wait** node "Wait" to delay for image generation completion.

24. **Retrieve Generated Image ID:**  
    - Add an **HTTP Request** node "Leo - Get imageId" to get the image ID.

25. **Download Generated Image:**  
    - Add an **HTTP Request** node "Download Image" to download the image file.

26. **Upload Image to WordPress:**  
    - Add an **HTTP Request** node "Upload media" to upload the image to WordPress media library.

27. **Associate Image with Post:**  
    - Add an **HTTP Request** node "Set image ID for the post" to set the featured image for the post.

28. **Notify Slack:**  
    - Add a **Slack** node "POST Blog Info" to send a notification about the published post.

29. **Update Airtable Status:**  
    - Add an **Airtable** node "UPDATE Status" to update the content status in Airtable.

30. **Connect Loop for Batch Processing:**  
    - Connect "UPDATE Status" back to "Loop Over Items" to continue batch processing.

31. **Configure Credentials:**  
    - Set up credentials for OpenAI, Airtable, WordPress REST API, Slack, and any external image generation services.

32. **Add Custom WordPress Code:**  
    - Update your WordPress theme’s `functions.php` with the provided snippet to enable automation compatibility.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                   |
|-------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Add a small update to your WordPress theme’s `functions.php` file to enable seamless automation. | See workflow description for code snippet instructions.                                         |
| Experiment with Airtable views or add filters for more granular control over your content pipeline. | Useful for customizing content selection and scheduling.                                        |
| Extend the workflow to include social media posting or analytics tracking.                       | Can be added by integrating respective APIs and nodes.                                         |
| For questions, refer to n8n documentation or reach out to the creator.                           | n8n docs: https://docs.n8n.io/                                                                   |
| Workflow uses OpenAI GPT, Airtable, WordPress REST API, RankMath SEO Plugin, Slack, and Leo API. | Ensure all API keys and credentials are correctly configured before running.                     |
| Workflow designed by AlexK1919, version 2.                                                      |                                                                                                 |

---

This comprehensive reference document enables advanced users and AI agents to fully understand, reproduce, and customize the **Ultimate Content Generator for WordPress** workflow efficiently.
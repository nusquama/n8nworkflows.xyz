Create AI-Generated Meta Ad Campaigns from Product URLs with OpenAI & Firecrawl

https://n8nworkflows.xyz/workflows/create-ai-generated-meta-ad-campaigns-from-product-urls-with-openai---firecrawl-8681


# Create AI-Generated Meta Ad Campaigns from Product URLs with OpenAI & Firecrawl

### 1. Workflow Overview

This workflow automates the creation of AI-generated meta advertising campaigns from product URLs. It is designed to receive product URLs via a form submission, scrape product data from those URLs, process the data through OpenAI language models to generate creative briefs and ad content, produce image or video creatives, and finally create and configure ad campaigns on Meta (Facebook) platforms.

The workflow is logically structured into the following blocks:

- **1.1 Input Reception & Data Scraping**: Receives product URLs via form input and scrapes content using Firecrawl.
- **1.2 AI Data Extraction & Processing**: Uses OpenAI language models to extract structured product data and generate creative briefs.
- **1.3 Ad Campaign Generation**: Generates ad campaign content and splits it for batch processing.
- **1.4 Creative Generation (Image/Video)**: Generates ad creatives (images or videos), uploads them to Meta, and prepares creative packets.
- **1.5 Campaign Setup & Execution**: Creates campaigns, ad sets, and ads on Meta, merging all creative elements and campaign info.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Data Scraping

- **Overview:**  
  This block handles the initial input reception from users via a form and scrapes the product URL content for further analysis.

- **Nodes Involved:**  
  - AI Ad Form Submission  
  - Scrape a url and get its content  

- **Node Details:**

  - **AI Ad Form Submission**  
    - *Type:* Form Trigger  
    - *Role:* Entry point for the workflow that listens for form submissions containing product URLs.  
    - *Configuration:* Uses a webhook to capture form data.  
    - *Connections:* Output goes to "Scrape a url and get its content".  
    - *Edge Cases:* Possible webhook timeouts or malformed form input.

  - **Scrape a url and get its content**  
    - *Type:* Firecrawl node  
    - *Role:* Scrapes HTML/content from the input URL to fetch product details.  
    - *Configuration:* Set to scrape the submitted URL.  
    - *Connections:* Passes the scraped content to "OpenAI extraction with JSON Schema".  
    - *Edge Cases:* Site blocking scraping, network timeouts, or unexpected page structures leading to incomplete data.

---

#### 1.2 AI Data Extraction & Processing

- **Overview:**  
  Extracts structured product information from the scraped content using OpenAI models, then generates a creative brief to inform ad generation.

- **Nodes Involved:**  
  - OpenAI extraction with JSON Schema  
  - Analyze Product1  
  - Merge1  
  - Creative Brief  

- **Node Details:**

  - **OpenAI extraction with JSON Schema**  
    - *Type:* LangChain Chain LLM node  
    - *Role:* Extracts structured JSON data from unstructured scraped content using an OpenAI model guided by a JSON schema.  
    - *Configuration:* Uses a schema to parse product info (e.g., title, description, features).  
    - *Connections:* Outputs to "Analyze Product1" and "Merge1".  
    - *Edge Cases:* Parsing errors if content format varies significantly; API rate limits or auth errors.

  - **Analyze Product1**  
    - *Type:* OpenAI LangChain node  
    - *Role:* Performs additional AI analysis on extracted product data to enrich or validate it.  
    - *Configuration:* Executes once per batch; likely uses a GPT model.  
    - *Connections:* Outputs to "Merge1".  
    - *Edge Cases:* Model hallucination or unexpected output format.

  - **Merge1**  
    - *Type:* Merge node  
    - *Role:* Combines outputs from extraction and analysis for unified processing.  
    - *Configuration:* Default merge mode.  
    - *Connections:* Passes combined data to "Creative Brief".  
    - *Edge Cases:* Potential data mismatch or empty inputs.

  - **Creative Brief**  
    - *Type:* OpenAI LangChain node  
    - *Role:* Generates a creative brief from merged product data to guide ad content generation.  
    - *Configuration:* Executes once to generate high-level creative guidance.  
    - *Connections:* Output leads to "Generate Ad Camp".  
    - *Edge Cases:* Model output inconsistency, API issues.

---

#### 1.3 Ad Campaign Generation

- **Overview:**  
  Generates ad campaign content (copy, headlines, descriptions) using OpenAI and prepares it for batch processing and platform adaptation.

- **Nodes Involved:**  
  - Generate Ad Camp  
  - Split  
  - Platform Adapter  
  - Batch  
  - Configuration Meta Ads  
  - Wait  

- **Node Details:**

  - **Generate Ad Camp**  
    - *Type:* LangChain Agent node  
    - *Role:* Uses AI to generate multiple ad campaign variations based on the creative brief.  
    - *Configuration:* Connected to multiple downstream nodes for splitting and parsing.  
    - *Connections:* Outputs to "Split".  
    - *Edge Cases:* AI output correctness, generation timeouts.

  - **Split**  
    - *Type:* Split Out node  
    - *Role:* Splits the generated ad campaigns into individual items for processing.  
    - *Connections:* Outputs to "Platform Adapter".  
    - *Edge Cases:* Empty splits or data loss.

  - **Platform Adapter**  
    - *Type:* Code node  
    - *Role:* Adapts or formats split data to Meta Ads platform requirements.  
    - *Connections:* Outputs to "Batch".  
    - *Edge Cases:* Script errors or unexpected data formats.

  - **Batch**  
    - *Type:* Split In Batches node  
    - *Role:* Processes campaigns in manageable batches to avoid API overload.  
    - *Connections:* Outputs to "Configuration Meta Ads" and "Wait".  
    - *Edge Cases:* Batch size misconfiguration, partial batch failures.

  - **Configuration Meta Ads**  
    - *Type:* Set node  
    - *Role:* Sets necessary parameters and configurations for Meta Ads API calls.  
    - *Connections:* Outputs to "Is it a Video?" node.  
    - *Edge Cases:* Missing or incorrect configuration values.

  - **Wait**  
    - *Type:* Wait node  
    - *Role:* Introduces delay control in processing, likely to handle rate limits or sequencing.  
    - *Connections:* Outputs to "Open AI Generate Image".  
    - *Edge Cases:* Timeout misconfiguration.

---

#### 1.4 Creative Generation (Image/Video)

- **Overview:**  
  Generates ad creatives (images or videos) using AI, uploads them to Meta, and prepares packets for campaign assembly.

- **Nodes Involved:**  
  - Open AI Generate Image  
  - B64 String to File  
  - Edit Fields  
  - Is it a Video?  
  - Upload Video to FB  
  - Set Video ID  
  - Create Video Creative  
  - Upload Image to FB  
  - Set Image Hash  
  - Create Image Creative  
  - Set Image Packet  
  - Set Video Packet  
  - Merge Creatives  

- **Node Details:**

  - **Open AI Generate Image**  
    - *Type:* HTTP Request node  
    - *Role:* Calls OpenAI (or related API) to generate base64-encoded images for ads.  
    - *Connections:* Outputs to "B64 String to File".  
    - *Edge Cases:* API errors, image generation failures.

  - **B64 String to File**  
    - *Type:* Convert To File node  
    - *Role:* Converts base64 string into file format usable by n8n and upload nodes.  
    - *Connections:* Outputs to "Edit Fields".  
    - *Edge Cases:* Conversion errors.

  - **Edit Fields**  
    - *Type:* Set node  
    - *Role:* Adjusts or sets fields in the data after image conversion, preparing for batch processing.  
    - *Connections:* Outputs to "Batch".  
    - *Edge Cases:* Field misconfiguration.

  - **Is it a Video?**  
    - *Type:* If node  
    - *Role:* Determines whether the creative is a video or an image to route processing accordingly.  
    - *Connections:* Outputs to either "Upload Video to FB" or "Upload Image to FB".  
    - *Edge Cases:* Incorrect media type detection.

  - **Upload Video to FB**  
    - *Type:* HTTP Request node  
    - *Role:* Uploads video files to Facebook.  
    - *Connections:* Outputs to "Set Video ID".  
    - *Edge Cases:* Upload failures, network errors, auth issues.

  - **Set Video ID**  
    - *Type:* Set node  
    - *Role:* Stores the uploaded video's ID for campaign use.  
    - *Connections:* Outputs to "Create Video Creative".  
    - *Edge Cases:* Missing video ID.

  - **Create Video Creative**  
    - *Type:* HTTP Request node  
    - *Role:* Creates the video ad creative on Meta using the uploaded video.  
    - *Connections:* Outputs to "Set Video Packet".  
    - *Edge Cases:* API errors.

  - **Upload Image to FB**  
    - *Type:* HTTP Request node  
    - *Role:* Uploads image files to Facebook.  
    - *Connections:* Outputs to "Set Image Hash".  
    - *Edge Cases:* Upload failures.

  - **Set Image Hash**  
    - *Type:* Set node  
    - *Role:* Stores the image hash returned by Facebook for creative use.  
    - *Connections:* Outputs to "Create Image Creative".  
    - *Edge Cases:* Missing hash.

  - **Create Image Creative**  
    - *Type:* HTTP Request node  
    - *Role:* Creates the image ad creative on Meta using the uploaded image hash.  
    - *Connections:* Outputs to "Set Image Packet".  
    - *Edge Cases:* API errors.

  - **Set Image Packet / Set Video Packet**  
    - *Type:* Set nodes  
    - *Role:* Prepare respective creative packets (image/video) for merging.  
    - *Connections:* Outputs to "Merge Creatives".  
    - *Edge Cases:* Data inconsistency.

  - **Merge Creatives**  
    - *Type:* Merge node  
    - *Role:* Combines image and video creative packets for unified campaign creation.  
    - *Connections:* Outputs to "Batch" and "Run Once".  
    - *Edge Cases:* Merge conflicts.

---

#### 1.5 Campaign Setup & Execution

- **Overview:**  
  Finalizes campaign creation by creating campaigns, ad sets, and ads on Meta, managing dependencies and IDs.

- **Nodes Involved:**  
  - Run Once  
  - Create Campaign  
  - Create Ad Set  
  - Save Adset Id  
  - Merge2  
  - Create Ad  

- **Node Details:**

  - **Run Once**  
    - *Type:* Function node  
    - *Role:* Ensures certain operations (e.g., campaign creation) are run only once per batch or overall workflow.  
    - *Connections:* Outputs to "Create Campaign".  
    - *Edge Cases:* Logic errors causing multiple runs.

  - **Create Campaign**  
    - *Type:* HTTP Request node  
    - *Role:* Calls Meta API to create the ad campaign entity.  
    - *Connections:* Outputs to "Create Ad Set".  
    - *Edge Cases:* API errors, missing campaign parameters.

  - **Create Ad Set**  
    - *Type:* HTTP Request node  
    - *Role:* Creates ad sets under the campaign with targeting and budget settings.  
    - *Connections:* Outputs to "Save Adset Id".  
    - *Edge Cases:* API errors, budget misconfiguration.

  - **Save Adset Id**  
    - *Type:* Set node  
    - *Role:* Saves the newly created ad set ID for linking ads.  
    - *Connections:* Outputs to "Merge2".  
    - *Edge Cases:* Missing ID or save errors.

  - **Merge2**  
    - *Type:* Merge node  
    - *Role:* Merges creative data with campaign data to prepare for ad creation.  
    - *Connections:* Outputs to "Create Ad".  
    - *Edge Cases:* Data mismatches.

  - **Create Ad**  
    - *Type:* HTTP Request node  
    - *Role:* Creates the actual ad entity on Meta using all prepared data packets.  
    - *Connections:* Final node, no outputs.  
    - *Edge Cases:* API errors, missing parameters.

---

### 3. Summary Table

| Node Name                  | Node Type                          | Functional Role                     | Input Node(s)                           | Output Node(s)                         | Sticky Note                 |
|----------------------------|----------------------------------|-----------------------------------|---------------------------------------|--------------------------------------|-----------------------------|
| AI Ad Form Submission       | Form Trigger                     | Entry point, receives product URLs| -                                     | Scrape a url and get its content     |                             |
| Scrape a url and get its content | Firecrawl node                 | Scrapes product page content      | AI Ad Form Submission                  | OpenAI extraction with JSON Schema   |                             |
| OpenAI extraction with JSON Schema | LangChain Chain LLM           | Extracts structured JSON data     | Scrape a url and get its content       | Analyze Product1, Merge1              |                             |
| Analyze Product1            | LangChain OpenAI Node            | Additional product analysis       | OpenAI extraction with JSON Schema     | Merge1                              |                             |
| Merge1                     | Merge Node                      | Merges extraction and analysis    | OpenAI extraction with JSON Schema, Analyze Product1 | Creative Brief                  |                             |
| Creative Brief             | LangChain OpenAI Node            | Generates creative brief          | Merge1                                | Generate Ad Camp                     |                             |
| Generate Ad Camp            | LangChain Agent Node             | Generates ad campaign variations  | Creative Brief                        | Split                              |                             |
| Split                      | Split Out Node                  | Splits campaigns for processing  | Generate Ad Camp                      | Platform Adapter                    |                             |
| Platform Adapter           | Code Node                      | Formats data for Meta Ads         | Split                                | Batch                              |                             |
| Batch                      | Split In Batches Node           | Batch processing of campaigns    | Platform Adapter, Edit Fields          | Configuration Meta Ads, Wait        |                             |
| Configuration Meta Ads     | Set Node                      | Sets Meta Ads parameters          | Batch                                | Is it a Video?                     |                             |
| Is it a Video?             | If Node                       | Routes image or video flow       | Configuration Meta Ads                | Upload Video to FB, Upload Image to FB |                             |
| Upload Video to FB          | HTTP Request Node              | Uploads video to Facebook         | Is it a Video? (True)                  | Set Video ID                       |                             |
| Set Video ID               | Set Node                      | Saves video ID                    | Upload Video to FB                    | Create Video Creative              |                             |
| Create Video Creative       | HTTP Request Node              | Creates video ad creative         | Set Video ID                         | Set Video Packet                  |                             |
| Upload Image to FB          | HTTP Request Node              | Uploads image to Facebook         | Is it a Video? (False)                 | Set Image Hash                    |                             |
| Set Image Hash             | Set Node                      | Saves image hash                  | Upload Image to FB                   | Create Image Creative             |                             |
| Create Image Creative       | HTTP Request Node              | Creates image ad creative         | Set Image Hash                      | Set Image Packet                 |                             |
| Set Image Packet           | Set Node                      | Prepares image creative packet    | Create Image Creative                | Merge Creatives                  |                             |
| Set Video Packet           | Set Node                      | Prepares video creative packet    | Create Video Creative                | Merge Creatives                  |                             |
| Merge Creatives            | Merge Node                    | Merges image and video creatives  | Set Image Packet, Set Video Packet   | Batch, Run Once                  |                             |
| Edit Fields                | Set Node                      | Adjusts fields after file conversion | B64 String to File                 | Batch                          |                             |
| B64 String to File         | Convert To File Node            | Converts base64 to file           | Open AI Generate Image               | Edit Fields                     |                             |
| Open AI Generate Image      | HTTP Request Node              | Generates images via OpenAI       | Wait                                | B64 String to File              |                             |
| Wait                       | Wait Node                     | Delays processing                 | Batch                               | Open AI Generate Image           |                             |
| Run Once                   | Function Node                 | Ensures single execution          | Merge Creatives                    | Create Campaign                  |                             |
| Create Campaign            | HTTP Request Node              | Creates Meta ad campaign          | Run Once                           | Create Ad Set                   |                             |
| Create Ad Set              | HTTP Request Node              | Creates Meta ad set               | Create Campaign                   | Save Adset Id                  |                             |
| Save Adset Id              | Set Node                      | Saves ad set ID                  | Create Ad Set                    | Merge2                        |                             |
| Merge2                     | Merge Node                    | Combines campaign and creative data | Save Adset Id, Create Ad Set       | Create Ad                      |                             |
| Create Ad                  | HTTP Request Node              | Creates the Meta ad               | Merge2                           | -                            |                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node** named "AI Ad Form Submission":  
   - Configure webhook to receive product URLs from a form submission.

2. **Add a Firecrawl node** "Scrape a url and get its content":  
   - Configure to scrape the URL received from the form input.

3. **Add a LangChain Chain LLM node** named "OpenAI extraction with JSON Schema":  
   - Configure with an OpenAI API credential.  
   - Use a JSON schema to parse the scraped content into structured product data.  
   - Connect input from the Firecrawl node.

4. **Create a LangChain OpenAI node** "Analyze Product1":  
   - Use GPT model to perform detailed product analysis.  
   - Set to execute once per batch.  
   - Connect input from "OpenAI extraction with JSON Schema".

5. **Add a Merge node** "Merge1":  
   - Merge outputs of "OpenAI extraction with JSON Schema" and "Analyze Product1".

6. **Add a LangChain OpenAI node** "Creative Brief":  
   - Generate a creative brief from merged data.  
   - Execute once.  
   - Connect input from "Merge1".

7. **Add a LangChain Agent node** "Generate Ad Camp":  
   - Generate ad campaign content variants.  
   - Connect input from "Creative Brief".

8. **Add a Split Out node** "Split":  
   - Split generated campaigns for parallel processing.  
   - Connect input from "Generate Ad Camp".

9. **Add a Code node** "Platform Adapter":  
   - Adapt split campaigns to Meta Ads format.  
   - Connect input from "Split".

10. **Add a Split In Batches node** "Batch":  
    - Set batch size per API rate limits or system constraints.  
    - Connect input from "Platform Adapter".

11. **Add a Set node** "Configuration Meta Ads":  
    - Set Meta Ads API-specific parameters (e.g., access tokens, campaign IDs).  
    - Connect input from "Batch".

12. **Add an If node** "Is it a Video?":  
    - Condition to check media type (video vs image).  
    - Connect input from "Configuration Meta Ads".

13. **Add HTTP Request nodes**:  
    - "Upload Video to FB" (for true branch)  
    - "Upload Image to FB" (for false branch)  
    - Configure with Meta OAuth2 credentials and appropriate endpoints.

14. **Add Set nodes**:  
    - "Set Video ID" connected after "Upload Video to FB"  
    - "Set Image Hash" connected after "Upload Image to FB"

15. **Add HTTP Request nodes**:  
    - "Create Video Creative" after "Set Video ID"  
    - "Create Image Creative" after "Set Image Hash"

16. **Add Set nodes**:  
    - "Set Video Packet" after "Create Video Creative"  
    - "Set Image Packet" after "Create Image Creative"

17. **Add Merge node** "Merge Creatives":  
    - Merge video and image packets.  
    - Connect outputs from "Set Video Packet" and "Set Image Packet".

18. **Add Set node** "Edit Fields":  
    - Adjust or set additional fields after image conversion (used after image generation).  
    - Connect input from "B64 String to File" (see below).

19. **Add HTTP Request node** "Open AI Generate Image":  
    - Use OpenAI or related API to generate images (base64).  
    - Connect input from "Wait".

20. **Add Convert To File node** "B64 String to File":  
    - Convert base64 images to file format for uploads.  
    - Connect input from "Open AI Generate Image".

21. **Connect "Edit Fields" output to "Batch" input** to loop creatives back into batch processing.

22. **Add Wait node** "Wait":  
    - Introduce controlled delay between batches or API calls.  
    - Connect input from "Batch".

23. **Add Function node** "Run Once":  
    - Ensures single execution for campaign creation steps.  
    - Connect input from "Merge Creatives".

24. **Add HTTP Request nodes for campaign setup**:  
    - "Create Campaign" connected from "Run Once"  
    - "Create Ad Set" connected from "Create Campaign"  
    - "Save Adset Id" (Set node) connected from "Create Ad Set"  
    - "Merge2" node merges "Save Adset Id" output and creative data  
    - "Create Ad" connected from "Merge2" finalizes ad creation.

25. **Set up all necessary credentials:**  
    - OpenAI API credentials for LangChain nodes and HTTP requests.  
    - Meta (Facebook) OAuth2 credentials for all Facebook API requests.

26. **Test with sample product URLs and monitor logs for errors or API limits.**

---

### 5. General Notes & Resources

| Note Content                                           | Context or Link                                  |
|--------------------------------------------------------|-------------------------------------------------|
| Workflow uses Firecrawl for efficient URL content scraping. | Firecrawl node documentation.                   |
| Utilizes LangChain nodes for structured OpenAI interaction. | https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-langchain/ |
| Meta Ads API requires proper OAuth2 credential setup.  | https://developers.facebook.com/docs/marketing-api/ |
| Includes batching and wait nodes to handle API rate limits. | n8n best practices for rate limiting.           |
| The workflow is suitable for e-commerce marketers seeking automated ad generation. | -                                               |

---

**Disclaimer:** The provided text is exclusively derived from an automated n8n workflow. It complies strictly with applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.
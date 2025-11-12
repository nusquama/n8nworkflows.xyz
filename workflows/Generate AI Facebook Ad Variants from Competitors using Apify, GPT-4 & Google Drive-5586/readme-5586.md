Generate AI Facebook Ad Variants from Competitors using Apify, GPT-4 & Google Drive

https://n8nworkflows.xyz/workflows/generate-ai-facebook-ad-variants-from-competitors-using-apify--gpt-4---google-drive-5586


# Generate AI Facebook Ad Variants from Competitors using Apify, GPT-4 & Google Drive

### 1. Workflow Overview

This workflow automates the generation of AI-powered Facebook ad variants inspired by competitors’ ads. It leverages Apify for scraping competitor ads, GPT-4 for generating creative spins on ad copy and images, and Google Drive for structured storage and management of generated assets. The workflow is tailored for digital marketers aiming to efficiently create multiple ad variants for testing and optimization.

The workflow is structured into these logical blocks:

- **1.1 Trigger and Data Input**: Manual trigger initiates the workflow, followed by reading input data from Google Drive and Google Sheets.
- **1.2 Competitor Ad Data Scraping**: Uses an HTTP request to Apify’s Ad Library Scraper API to fetch competitor ad data.
- **1.3 Data Filtering and Limiting**: Filters and limits scraped data to manageable subsets.
- **1.4 Folder Structure Creation on Google Drive**: Creates a hierarchical folder system for storing raw and processed ad content.
- **1.5 AI-Powered Ad Copy Generation**: Uses OpenAI GPT-4 nodes to spin competitor ad copy into new variants.
- **1.6 Image Generation and Download**: Downloads competitor ad images, generates AI images using GPT Image API, and converts images into files.
- **1.7 Asset Upload and Logging**: Uploads generated assets to Google Drive and logs details into Google Sheets.
- **1.8 Batch Processing and Workflow Control**: Manages looping over multiple ad variants and pacing execution with wait nodes.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger and Data Input

- **Overview:**  
Starts the workflow manually and loads initial data from Google Drive and Google Sheets, serving as the foundation for subsequent processing.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’  
  - Google Drive  
  - Google Sheets1  
  - Edit Fields1  
  - Google Sheets2

- **Node Details:**  
  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Starts the workflow on user command.  
    - Inputs: None  
    - Outputs: Triggers Google Drive node.  
    - Edge Cases: None typical; manual trigger requires user action.  
  - **Google Drive**  
    - Type: Google Drive node  
    - Role: Reads input files or folders from Google Drive that contain competitor or configuration data.  
    - Configuration: Likely set to list or read files/folders from a predefined Drive path.  
    - Inputs: From Manual Trigger  
    - Outputs: To Google Sheets1  
    - Edge Cases: Authentication issues, missing files, permission errors.  
  - **Google Sheets1**  
    - Type: Google Sheets node  
    - Role: Reads structured competitor data or metadata from a Google Sheet.  
    - Configuration: Reads specific spreadsheet and worksheet.  
    - Inputs: From Google Drive  
    - Outputs: To Edit Fields1  
    - Edge Cases: Sheet access errors, empty cells, rate limits.  
  - **Edit Fields1**  
    - Type: Set node  
    - Role: Modifies or formats data fields read from Sheets for downstream compatibility.  
    - Inputs: From Google Sheets1  
    - Outputs: To Google Sheets2  
    - Edge Cases: Expression failures if fields missing.  
  - **Google Sheets2**  
    - Type: Google Sheets node  
    - Role: Possibly writes or further processes the edited data.  
    - Inputs: From Edit Fields1  
    - Outputs: Initiates next processing blocks or stores intermediate data.  
    - Edge Cases: Write permissions, rate limiting.

---

#### 1.2 Competitor Ad Data Scraping

- **Overview:**  
Fetches competitor Facebook ad data from the Apify Ad Library Scraper API via an HTTP request node.

- **Nodes Involved:**  
  - Set Variables  
  - Run Ad Library Scraper  
  - Filter  
  - Limit

- **Node Details:**  
  - **Set Variables**  
    - Type: Set node  
    - Role: Prepares variables such as API keys, search parameters, or URLs needed for scraping.  
    - Inputs: From Google Sheets2 or prior data  
    - Outputs: To Run Ad Library Scraper  
    - Edge Cases: Missing or invalid variables causing API request failures.  
  - **Run Ad Library Scraper**  
    - Type: HTTP Request  
    - Role: Calls Apify’s Facebook Ad Library Scraper to retrieve competitor ads.  
    - Configuration: HTTP GET/POST with API URL, authentication headers, query parameters.  
    - Inputs: From Set Variables  
    - Outputs: To Filter  
    - Edge Cases: HTTP errors, timeout, invalid responses, API quota limits.  
  - **Filter**  
    - Type: Filter node  
    - Role: Filters scraped ads based on criteria (e.g., relevance, date, ad type).  
    - Inputs: From Run Ad Library Scraper  
    - Outputs: To Limit  
    - Edge Cases: No data passing filter, filter expression errors.  
  - **Limit**  
    - Type: Limit node  
    - Role: Restricts the number of ads processed downstream to control workflow load.  
    - Inputs: From Filter  
    - Outputs: To Create Asset Parent Folder  
    - Edge Cases: Limit set too low or high impacting output volume.

---

#### 1.3 Folder Structure Creation on Google Drive

- **Overview:**  
Creates a multi-level folder structure on Google Drive to organize downloaded and generated ad assets.

- **Nodes Involved:**  
  - Create Asset Parent Folder  
  - Create Child Source Folder  
  - Create Child Spun Folder

- **Node Details:**  
  - **Create Asset Parent Folder**  
    - Type: Google Drive node  
    - Role: Creates the top-level folder for storing this batch of assets.  
    - Inputs: From Limit  
    - Outputs: To Create Child Source Folder  
    - Edge Cases: Folder already exists, permission errors.  
  - **Create Child Source Folder**  
    - Type: Google Drive node  
    - Role: Creates a subfolder for raw source assets (e.g., scraped competitor ads).  
    - Inputs: From Create Asset Parent Folder  
    - Outputs: To Create Child Spun Folder  
    - Edge Cases: Same as above.  
  - **Create Child Spun Folder**  
    - Type: Google Drive node  
    - Role: Creates a subfolder for AI-generated spun ad variants.  
    - Inputs: From Create Child Source Folder  
    - Outputs: To Download Static Image Ad  
    - Edge Cases: Same as above.

---

#### 1.4 AI-Powered Ad Copy Generation

- **Overview:**  
Leverages OpenAI GPT-4 via Langchain nodes to generate creative variations (“spin”) of competitor ad copy.

- **Nodes Involved:**  
  - Google Drive2  
  - OpenAI  
  - Spin Prompts  
  - Split Out  
  - Edit Fields  
  - Loop Over Items

- **Node Details:**  
  - **Google Drive2**  
    - Type: Google Drive node  
    - Role: Reads input files containing competitor ad text or prompts for AI.  
    - Inputs: From Google Drive1  
    - Outputs: To OpenAI  
    - Edge Cases: File not found, access errors.  
  - **OpenAI**  
    - Type: OpenAI Langchain node  
    - Role: Sends competitor ad text to GPT-4 for initial processing (e.g., summarization).  
    - Configuration: Uses GPT-4 model, with prompt templates.  
    - Inputs: From Google Drive2  
    - Outputs: To Spin Prompts  
    - Edge Cases: API quota, network errors, prompt generation failures.  
  - **Spin Prompts**  
    - Type: OpenAI Langchain node  
    - Role: Generates multiple spun versions of ad copy.  
    - Inputs: From OpenAI  
    - Outputs: To Split Out  
    - Edge Cases: As above.  
  - **Split Out**  
    - Type: Split Out node  
    - Role: Separates multiple generated texts into individual items for processing.  
    - Inputs: From Spin Prompts  
    - Outputs: To Edit Fields  
    - Edge Cases: Empty outputs, splitting errors.  
  - **Edit Fields**  
    - Type: Set node  
    - Role: Formats or adds metadata to spun ad copies before batch processing.  
    - Inputs: From Split Out  
    - Outputs: To Loop Over Items  
    - Edge Cases: Expression failures.  
  - **Loop Over Items**  
    - Type: Split In Batches node  
    - Role: Loops through each spun ad variant for image generation and storage.  
    - Inputs: From Edit Fields  
    - Outputs: To Download Static Image Ad1 (and loops back)  
    - Edge Cases: Large number of items causing slowdowns.

---

#### 1.5 Image Generation and Download

- **Overview:**  
Downloads competitor ad images and generates new AI images inspired by them, converting and storing them properly.

- **Nodes Involved:**  
  - Download Static Image Ad  
  - Google Drive1  
  - Download Static Image Ad1  
  - Generate Image Using GPT Image 1  
  - Convert to File  
  - Google Drive3

- **Node Details:**  
  - **Download Static Image Ad**  
    - Type: HTTP Request node  
    - Role: Downloads static images from competitor ads.  
    - Inputs: From Create Child Spun Folder  
    - Outputs: To Google Drive1  
    - Edge Cases: URL invalid, timeout, large file size.  
  - **Google Drive1**  
    - Type: Google Drive node  
    - Role: Uploads downloaded static images into the “source” folder on Drive.  
    - Inputs: From Download Static Image Ad  
    - Outputs: To Google Drive2 (for ad text processing)  
    - Edge Cases: Upload failures, quota limits.  
  - **Download Static Image Ad1**  
    - Type: HTTP Request node  
    - Role: Downloads or fetches additional static images per batch item.  
    - Inputs: From Loop Over Items (2nd output)  
    - Outputs: To Generate Image Using GPT Image 1  
    - Edge Cases: Same as above.  
  - **Generate Image Using GPT Image 1**  
    - Type: HTTP Request node  
    - Role: Calls GPT Image generation API to create AI images based on prompts or competitor images.  
    - Configuration: Executes once per item.  
    - Inputs: From Download Static Image Ad1  
    - Outputs: To Convert to File  
    - Edge Cases: API limits, generation failures, slow response.  
  - **Convert to File**  
    - Type: Convert To File node  
    - Role: Converts raw image data into file format suitable for Google Drive upload.  
    - Inputs: From Generate Image Using GPT Image 1  
    - Outputs: To Google Drive3  
    - Edge Cases: Conversion failures, unsupported formats.  
  - **Google Drive3**  
    - Type: Google Drive node  
    - Role: Uploads generated AI images into the “spun” folder on Drive.  
    - Inputs: From Convert to File  
    - Outputs: To Google Sheets (logging)  
    - Edge Cases: Upload failures.

---

#### 1.6 Asset Upload and Logging

- **Overview:**  
Logs generated assets' metadata into Google Sheets and manages pacing of the workflow.

- **Nodes Involved:**  
  - Google Sheets  
  - Wait

- **Node Details:**  
  - **Google Sheets**  
    - Type: Google Sheets node  
    - Role: Writes metadata (file links, ad copy variants) to a Google Sheet for tracking.  
    - Inputs: From Google Drive3  
    - Outputs: To Wait  
    - Edge Cases: Write permission errors, rate limits.  
  - **Wait**  
    - Type: Wait node  
    - Role: Pauses workflow briefly to manage API rate limits or pacing.  
    - Inputs: From Google Sheets  
    - Outputs: Loops back to Loop Over Items for next batch processing.  
    - Edge Cases: Incorrect wait time causing delays or throttling.

---

### 3. Summary Table

| Node Name                   | Node Type                        | Functional Role                     | Input Node(s)                | Output Node(s)             | Sticky Note                                              |
|-----------------------------|---------------------------------|-----------------------------------|-----------------------------|----------------------------|----------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                  | Starts the workflow                | -                           | Google Drive               |                                                          |
| Google Drive                | Google Drive                    | Reads input files/folders         | When clicking ‘Execute workflow’ | Google Sheets1             |                                                          |
| Google Sheets1             | Google Sheets                   | Reads competitor metadata         | Google Drive                | Edit Fields1               |                                                          |
| Edit Fields1               | Set                            | Formats competitor data           | Google Sheets1              | Google Sheets2             |                                                          |
| Google Sheets2             | Google Sheets                   | Further processes data            | Edit Fields1                | Set Variables              |                                                          |
| Set Variables              | Set                            | Prepares scraping variables       | Google Sheets2              | Run Ad Library Scraper     |                                                          |
| Run Ad Library Scraper     | HTTP Request                   | Calls Apify API for competitor ads| Set Variables               | Filter                    |                                                          |
| Filter                    | Filter                         | Filters ads based on criteria     | Run Ad Library Scraper      | Limit                     |                                                          |
| Limit                     | Limit                          | Limits number of ads processed    | Filter                     | Create Asset Parent Folder |                                                          |
| Create Asset Parent Folder | Google Drive                   | Creates main asset folder         | Limit                      | Create Child Source Folder |                                                          |
| Create Child Source Folder | Google Drive                   | Creates source assets folder      | Create Asset Parent Folder  | Create Child Spun Folder   |                                                          |
| Create Child Spun Folder   | Google Drive                   | Creates spun assets folder        | Create Child Source Folder  | Download Static Image Ad   |                                                          |
| Download Static Image Ad   | HTTP Request                   | Downloads competitor images       | Create Child Spun Folder    | Google Drive1              |                                                          |
| Google Drive1              | Google Drive                   | Uploads source images             | Download Static Image Ad    | Google Drive2              |                                                          |
| Google Drive2              | Google Drive                   | Reads ad text files for AI input  | Google Drive1              | OpenAI                    |                                                          |
| OpenAI                    | OpenAI Langchain               | Initial GPT-4 processing          | Google Drive2              | Spin Prompts              |                                                          |
| Spin Prompts              | OpenAI Langchain               | Generates spun ad variants        | OpenAI                    | Split Out                 |                                                          |
| Split Out                 | Split Out                      | Separates multiple variants       | Spin Prompts              | Edit Fields               |                                                          |
| Edit Fields               | Set                            | Formats spun ad data              | Split Out                 | Loop Over Items           |                                                          |
| Loop Over Items           | Split In Batches               | Iterates over ad variants         | Edit Fields               | Download Static Image Ad1 (and loops back) |                                                          |
| Download Static Image Ad1 | HTTP Request                   | Downloads additional images       | Loop Over Items (2nd output) | Generate Image Using GPT Image 1 |                                                          |
| Generate Image Using GPT Image 1 | HTTP Request                   | Calls GPT Image generation API    | Download Static Image Ad1 | Convert to File           |                                                          |
| Convert to File           | Convert To File                | Converts image data to file       | Generate Image Using GPT Image 1 | Google Drive3             |                                                          |
| Google Drive3             | Google Drive                   | Uploads AI-generated images       | Convert to File            | Google Sheets             |                                                          |
| Google Sheets             | Google Sheets                   | Logs asset metadata               | Google Drive3              | Wait                      |                                                          |
| Wait                      | Wait                          | Controls pacing and rate limits   | Google Sheets              | Loop Over Items           |                                                          |
| Edit Fields1              | Set                            | Formats initial input data        | Google Sheets1             | Google Sheets2             |                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**:  
   - Node Type: Manual Trigger  
   - No parameters needed. This node starts the workflow.

2. **Add Google Drive Node (Input)**:  
   - Node Type: Google Drive  
   - Configure with OAuth2 credentials to your Google Drive.  
   - Set to read input files or folders where competitor or configuration data is stored.

3. **Add Google Sheets Node (Read Competitor Metadata)**:  
   - Node Type: Google Sheets  
   - Configure to read a specific spreadsheet and worksheet containing competitor ad info.  
   - Use OAuth2 credentials for Google Sheets.  
   - Connect from Google Drive node.

4. **Add Set Node (Edit Fields1)**:  
   - Modify or rename fields fetched from Google Sheets for downstream compatibility.  
   - Connect from Google Sheets1.

5. **Add Google Sheets Node (Further Processing)**:  
   - Possibly to write or prepare structured data for scraping parameters.  
   - Connect from Edit Fields1.

6. **Add Set Node (Set Variables)**:  
   - Define variables such as API keys, search queries, or URLs for Apify API.  
   - Connect from Google Sheets2.

7. **Add HTTP Request Node (Run Ad Library Scraper)**:  
   - Configure to call Apify Facebook Ad Library Scraper API.  
   - Set method, URL, headers including API key, and query parameters as per Apify docs.  
   - Connect from Set Variables.

8. **Add Filter Node**:  
   - Define filter rules to refine the scraped ad data, e.g., filter ads by date or type.  
   - Connect from Run Ad Library Scraper.

9. **Add Limit Node**:  
   - Set a maximum number of items to process downstream to control workflow size.  
   - Connect from Filter.

10. **Add Three Google Drive Nodes for Folder Creation**:  
    - Create Asset Parent Folder (top level folder).  
    - Create Child Source Folder (for raw ads).  
    - Create Child Spun Folder (for AI-generated variants).  
    - Connect sequentially: Limit → Parent Folder → Child Source → Child Spun.

11. **Add HTTP Request Node (Download Static Image Ad)**:  
    - Downloads competitor ad images.  
    - Connect from Create Child Spun Folder.

12. **Add Google Drive Node (Upload Source Images)**:  
    - Upload images downloaded to “source” folder.  
    - Connect from Download Static Image Ad.

13. **Add Google Drive Node (Read Ad Text Files)**:  
    - Read ad texts or prompts for AI.  
    - Connect from Google Drive1.

14. **Add OpenAI Node (Initial Processing)**:  
    - Configure with OpenAI GPT-4 credentials.  
    - Set prompt templates to process competitor ad copy.  
    - Connect from Google Drive2.

15. **Add OpenAI Node (Spin Prompts)**:  
    - Further spins or generates multiple ad copy variants.  
    - Connect from OpenAI.

16. **Add Split Out Node**:  
    - Splits multi-result text outputs into individual items.  
    - Connect from Spin Prompts.

17. **Add Set Node (Edit Fields)**:  
    - Format or annotate spun text items with metadata.  
    - Connect from Split Out.

18. **Add Split In Batches Node (Loop Over Items)**:  
    - Iterate over each spun ad variant for further processing.  
    - Connect from Edit Fields.

19. **Add HTTP Request Node (Download Static Image Ad1)**:  
    - Download additional images per ad variant.  
    - Connect from Loop Over Items (2nd output).

20. **Add HTTP Request Node (Generate Image Using GPT Image 1)**:  
    - Call GPT Image generation API.  
    - Configure API endpoint, auth, and parameters to generate images based on prompts.  
    - Set execute once per batch item.  
    - Connect from Download Static Image Ad1.

21. **Add Convert To File Node**:  
    - Convert generated image data into file format suitable for Google Drive upload.  
    - Connect from Generate Image Using GPT Image 1.

22. **Add Google Drive Node (Upload AI-Generated Images)**:  
    - Upload converted images to “spun” folder.  
    - Connect from Convert to File.

23. **Add Google Sheets Node (Log Metadata)**:  
    - Write metadata about generated assets to a spreadsheet for tracking.  
    - Connect from Google Drive3.

24. **Add Wait Node**:  
    - Configure wait time to manage rate limits or pacing.  
    - Connect from Google Sheets.

25. **Loop Control**:  
    - Connect Wait node back to Loop Over Items node input to continue processing batches.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                       |
|-------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| This workflow requires valid OAuth2 credentials for Google Drive and Google Sheets nodes.       | Google API Console for enabling Drive and Sheets APIs.                                              |
| Apify Facebook Ad Library Scraper API key is needed for scraping competitor ads.                 | https://apify.com/store/facebook-ad-library-scraper                                                  |
| OpenAI GPT-4 API key with Langchain integration is required for AI text and image generation.   | https://platform.openai.com/docs/models/gpt-4                                                        |
| Proper rate limiting and wait times are essential to avoid API throttling or quota exhaustion.   |                                                                                                     |
| The folder structure on Google Drive helps keep raw and processed assets organized systematically.|                                                                                                     |

---

This document fully covers the workflow’s structure, node functionality, dependencies, and reproduction instructions, enabling advanced users and automation agents to understand, maintain, and extend the workflow effectively.
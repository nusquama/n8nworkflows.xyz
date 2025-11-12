Scrape Competitor Reviews & Generate Ad Creatives with Bright Data & OpenAI

https://n8nworkflows.xyz/workflows/scrape-competitor-reviews---generate-ad-creatives-with-bright-data---openai-3624


# Scrape Competitor Reviews & Generate Ad Creatives with Bright Data & OpenAI

### 1. Workflow Overview

This workflow automates the process of scraping competitor product reviews from Amazon using Bright Data, analyzing the aggregated reviews with OpenAI’s GPT-4o mini model, generating a creative ad image based on identified competitor weaknesses, and distributing the results via email to media buyers. It is designed for marketing teams aiming to leverage competitor insights for targeted ad creative development.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures the Amazon product URL via a form submission.
- **1.2 Bright Data Scraping Trigger:** Initiates the scraping job on Bright Data using the provided URL.
- **1.3 Scraping Status Polling:** Periodically checks the scraping job status until completion.
- **1.4 Data Retrieval:** Downloads the scraped reviews in JSON format.
- **1.5 Data Storage:** Appends the scraped reviews into a Google Sheets document for record-keeping.
- **1.6 Review Aggregation:** Combines all review texts into a single aggregated string.
- **1.7 Review Analysis with OpenAI:** Uses GPT-4o mini to summarize competitor weaknesses from aggregated reviews.
- **1.8 Creative Image Generation:** Requests OpenAI to generate a 1080x1080 ad image addressing the identified pain points.
- **1.9 Email Distribution:** Sends the summary and generated image via Gmail to the media buying team.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Captures the Amazon product URL from the user via a form trigger to start the workflow.

- **Nodes Involved:**  
  - On form submission - Amazon Reviews

- **Node Details:**  
  - **On form submission - Amazon Reviews**  
    - Type: Form Trigger  
    - Role: Entry point capturing user input.  
    - Configuration: Single required field labeled "Amazon Product URL" with placeholder example URL.  
    - Input: User-submitted form data.  
    - Output: JSON containing the Amazon product URL.  
    - Edge Cases: Missing or malformed URL input could cause downstream API call failures. Validation is minimal; user must input a valid Amazon product URL.  
    - No sub-workflow.

---

#### 1.2 Bright Data Scraping Trigger

- **Overview:**  
  Sends a POST request to Bright Data’s API to trigger scraping of Amazon reviews for the provided URL.

- **Nodes Involved:**  
  - HTTP Request- Post API call to Bright Data  
  - Sticky Note1 (instructional)

- **Node Details:**  
  - **HTTP Request- Post API call to Bright Data**  
    - Type: HTTP Request (POST)  
    - Role: Initiates Bright Data scraping dataset with the Amazon URL.  
    - Configuration:  
      - URL: `https://api.brightdata.com/datasets/v3/trigger`  
      - Method: POST  
      - Body: JSON array with the URL from form input (`{{ $json['Amazon Product URL'] }}`)  
      - Query Parameters: dataset_id fixed to `gd_le8e811kzy4ggddlq`, include_errors=true  
      - Headers: Authorization Bearer token (Bright Data API key)  
    - Input: JSON from form trigger node.  
    - Output: JSON containing snapshot_id for the scraping job.  
    - Edge Cases: Authentication errors if API key invalid; malformed URL may cause scraping failure; API rate limits.  
    - Sticky Note1 reminds user to add competitor Amazon link here.

---

#### 1.3 Scraping Status Polling

- **Overview:**  
  Waits for a fixed interval, then polls Bright Data API to check if the scraping snapshot is complete.

- **Nodes Involved:**  
  - Wait - Polling Bright Data  
  - Snapshot Progress  
  - If - Checking status of Snapshot - if data is ready or not

- **Node Details:**  
  - **Wait - Polling Bright Data**  
    - Type: Wait  
    - Role: Pauses workflow for 1 minute before polling status.  
    - Configuration: Wait 1 minute, repeats until condition met.  
    - Input: Output from Bright Data trigger.  
    - Output: Passes data to Snapshot Progress node.  
    - Edge Cases: Too short wait may cause excessive API calls; too long delays workflow.  
  - **Snapshot Progress**  
    - Type: HTTP Request (GET)  
    - Role: Queries Bright Data API for scraping job progress.  
    - Configuration:  
      - URL uses snapshot_id from previous POST response.  
      - Header Authorization with Bright Data API key.  
    - Input: snapshot_id from POST node.  
    - Output: JSON with status field (e.g., "running", "completed").  
    - Edge Cases: API errors, invalid snapshot_id, network timeouts.  
  - **If - Checking status of Snapshot - if data is ready or not**  
    - Type: Conditional (If)  
    - Role: Checks if status equals "running" to decide whether to wait again or proceed.  
    - Configuration: Condition: `{{$json.status}} == "running"`  
    - Input: Output from Snapshot Progress node.  
    - Output:  
      - True branch: loops back to Wait node to poll again.  
      - False branch: proceeds to data retrieval.  
    - Edge Cases: Unexpected status values; infinite loops if status never changes.

---

#### 1.4 Data Retrieval

- **Overview:**  
  Retrieves the completed scraping snapshot data in JSON format from Bright Data.

- **Nodes Involved:**  
  - HTTP Request - Getting data from Bright Data

- **Node Details:**  
  - **HTTP Request - Getting data from Bright Data**  
    - Type: HTTP Request (GET)  
    - Role: Downloads scraped review data as JSON.  
    - Configuration:  
      - URL uses snapshot_id from initial POST node.  
      - Query parameter: format=json  
      - Header Authorization with Bright Data API key.  
    - Input: snapshot_id from POST node.  
    - Output: JSON array of scraped reviews.  
    - Edge Cases: API errors, incomplete data if called prematurely, network issues.

---

#### 1.5 Data Storage

- **Overview:**  
  Appends the retrieved review data into a Google Sheets document for persistent storage and further analysis.

- **Nodes Involved:**  
  - Google Sheets - Adding All Reviews  
  - Sticky Note10 (instructions about Google Sheets template)

- **Node Details:**  
  - **Google Sheets - Adding All Reviews**  
    - Type: Google Sheets (Append)  
    - Role: Adds each review as a new row in a predefined Google Sheet.  
    - Configuration:  
      - Document ID: fixed to the provided template sheet ID.  
      - Sheet Name: gid=0 (first sheet)  
      - Columns: Auto-mapped from input JSON fields (e.g., url, product_name, review_text, rating, author_name, etc.)  
      - Credentials: Google Sheets OAuth2 account configured.  
    - Input: JSON array of reviews from Bright Data data retrieval node.  
    - Output: Confirmation of append operation.  
    - Edge Cases: Google Sheets API quota limits, authentication errors, schema mismatch if data fields change.  
    - Sticky Note10 provides link to Google Sheets template and setup instructions.

---

#### 1.6 Review Aggregation

- **Overview:**  
  Aggregates all review texts into a single concatenated string to prepare for AI analysis.

- **Nodes Involved:**  
  - Aggregating all reviews

- **Node Details:**  
  - **Aggregating all reviews**  
    - Type: Aggregate  
    - Role: Concatenates all `review_text` fields into one aggregated field named `Aggregated_reviews`.  
    - Configuration: Aggregation on field `review_text` with rename to `Aggregated_reviews`.  
    - Input: Data from Google Sheets append node (or directly from Bright Data JSON).  
    - Output: Single JSON object with aggregated reviews text.  
    - Edge Cases: Very large text size may exceed LLM input limits; empty or missing review_text fields.

---

#### 1.7 Review Analysis with OpenAI

- **Overview:**  
  Sends the aggregated reviews to OpenAI GPT-4o mini to generate a summary highlighting competitors’ main weaknesses.

- **Nodes Involved:**  
  - Basic LLM Chain - Summary of Recent reviews  
  - OpenAI Chat Model  
  - Sticky Note4 (prompt adjustment instructions)

- **Node Details:**  
  - **Basic LLM Chain - Summary of Recent reviews**  
    - Type: LangChain LLM Chain  
    - Role: Defines prompt and sends aggregated reviews for summarization.  
    - Configuration:  
      - Prompt: Reads aggregated reviews and instructs to summarize weakest points without naming competitors.  
      - Input variable: `{{ $json.Aggregated_reviews }}`  
    - Input: Aggregated reviews text.  
    - Output: Summary text of competitor weaknesses.  
    - Edge Cases: LLM API errors, prompt misinterpretation, incomplete input data.  
  - **OpenAI Chat Model**  
    - Type: OpenAI Chat Model (GPT-4o mini)  
    - Role: Executes the LLM call defined by the chain node.  
    - Configuration: Model set to "gpt-4o-mini".  
    - Credentials: OpenAI API key configured.  
    - Input: Prompt from LLM Chain node.  
    - Output: Text summary.  
    - Edge Cases: API rate limits, authentication errors, network issues.  
  - Sticky Note4 advises users to customize the prompt with company info or adjust summary requirements.

---

#### 1.8 Creative Image Generation

- **Overview:**  
  Requests OpenAI to generate a 1080x1080 ad image that visually addresses the main pain points identified in the review summary.

- **Nodes Involved:**  
  - OpenAI- Generating image

- **Node Details:**  
  - **OpenAI- Generating image**  
    - Type: LangChain OpenAI Image Generation  
    - Role: Generates an ad creative image based on prompt parameters.  
    - Configuration:  
      - Prompt JSON specifies:  
        - Dimensions: 1080x1080  
        - Target audience: B2C customer  
        - Pain points source: selects one pain point from the summary text  
        - Style: weird-and-fun with weird objects like "Fruit with Faces" and "Realistic People"  
        - Focus: address biggest pain point, avoid competitor names  
        - Background colors: bold red and black  
        - Headline: no text headline or other text  
      - Resource: image generation  
    - Input: Summary text from previous node.  
    - Output: Binary image data.  
    - Credentials: OpenAI API key configured.  
    - Edge Cases: Image generation API limits, prompt parsing errors, large image size handling.

---

#### 1.9 Email Distribution

- **Overview:**  
  Sends an email with the review summary and the generated ad creative image attached to the media buying team.

- **Nodes Involved:**  
  - Gmail - Sending creative to Media Buyers

- **Node Details:**  
  - **Gmail - Sending creative to Media Buyers**  
    - Type: Gmail node (OAuth2)  
    - Role: Sends email with text and image attachment.  
    - Configuration:  
      - Recipient: yaron.been@gmail.com (can be customized)  
      - Subject: Includes current timestamp and static text referencing competitor analysis  
      - Message body: Includes date, summary text from LLM, and note about attached image for Meta ads  
      - Attachment: Binary image from OpenAI image generation node  
      - Email type: plain text  
    - Credentials: Gmail OAuth2 account configured.  
    - Input: Summary text and binary image.  
    - Output: Email send confirmation.  
    - Edge Cases: Gmail API quota limits, authentication errors, attachment size limits.

---

### 3. Summary Table

| Node Name                                | Node Type                         | Functional Role                          | Input Node(s)                                | Output Node(s)                                | Sticky Note                                                                                                                                    |
|-----------------------------------------|----------------------------------|----------------------------------------|----------------------------------------------|-----------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------|
| On form submission - Amazon Reviews     | Form Trigger                     | Captures Amazon product URL input      | None                                         | HTTP Request- Post API call to Bright Data    |                                                                                                                                                |
| HTTP Request- Post API call to Bright Data | HTTP Request (POST)              | Triggers Bright Data scraping job      | On form submission - Amazon Reviews          | Wait - Polling Bright Data                     | Sticky Note1: Add your competitors Amazon link here.                                                                                           |
| Wait - Polling Bright Data               | Wait                            | Waits 1 minute before polling status   | HTTP Request- Post API call to Bright Data   | Snapshot Progress                              |                                                                                                                                                |
| Snapshot Progress                        | HTTP Request (GET)               | Checks scraping job status              | Wait - Polling Bright Data                    | If - Checking status of Snapshot               | Sticky Note9: Workflow assistance and support contact info, links to YouTube, LinkedIn, Bright Data docs.                                      |
| If - Checking status of Snapshot         | If (Conditional)                 | Determines if scraping is complete      | Snapshot Progress                             | Wait - Polling Bright Data (if running)       |                                                                                                                                                |
|                                         |                                  |                                        |                                               | HTTP Request - Getting data from Bright Data (if complete) |                                                                                                                                                |
| HTTP Request - Getting data from Bright Data | HTTP Request (GET)               | Retrieves scraped review data           | If - Checking status of Snapshot              | Google Sheets - Adding All Reviews             |                                                                                                                                                |
| Google Sheets - Adding All Reviews       | Google Sheets                   | Appends reviews to Google Sheets       | HTTP Request - Getting data from Bright Data | Aggregating all reviews                         | Sticky Note10: Google Sheets template link and setup instructions.                                                                             |
| Aggregating all reviews                   | Aggregate                      | Concatenates all review texts           | Google Sheets - Adding All Reviews            | Basic LLM Chain - Summary of Recent reviews    |                                                                                                                                                |
| Basic LLM Chain - Summary of Recent reviews | LangChain LLM Chain             | Summarizes competitor weaknesses       | Aggregating all reviews                        | OpenAI Chat Model                              | Sticky Note4: Prompt adjustment instructions for company info and summary customization.                                                      |
| OpenAI Chat Model                        | OpenAI Chat Model               | Executes LLM summarization              | Basic LLM Chain - Summary of Recent reviews  | OpenAI- Generating image                        |                                                                                                                                                |
| OpenAI- Generating image                  | LangChain OpenAI Image          | Generates ad creative image             | OpenAI Chat Model                             | Gmail - Sending creative to Media Buyers       |                                                                                                                                                |
| Gmail - Sending creative to Media Buyers | Gmail                          | Sends email with summary and image     | OpenAI- Generating image                       | None                                          |                                                                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node:**  
   - Type: Form Trigger  
   - Configure form title: "Please Paste The URL of the Amazon Product"  
   - Add one required field: "Amazon Product URL" with placeholder example URL.  
   - This node will receive the competitor Amazon product URL.

2. **Add an HTTP Request node to trigger Bright Data scraping:**  
   - Type: HTTP Request (POST)  
   - URL: `https://api.brightdata.com/datasets/v3/trigger`  
   - Method: POST  
   - Body (JSON): `[{"url": "{{ $json['Amazon Product URL'] }}"}]`  
   - Query Parameters:  
     - dataset_id = `gd_le8e811kzy4ggddlq`  
     - include_errors = `true`  
   - Headers: Authorization Bearer `<YOUR_BRIGHT_DATA_API_KEY>`  
   - Connect Form Trigger output to this node input.

3. **Add a Wait node for polling:**  
   - Type: Wait  
   - Duration: 1 minute  
   - Connect HTTP Request (Bright Data trigger) output to Wait node input.

4. **Add HTTP Request node to check scraping progress:**  
   - Type: HTTP Request (GET)  
   - URL: `https://api.brightdata.com/datasets/v3/progress/{{ $('HTTP Request- Post API call to Bright Data').item.json.snapshot_id }}`  
   - Headers: Authorization Bearer `<YOUR_BRIGHT_DATA_API_KEY>`  
   - Connect Wait node output to this node input.

5. **Add an If node to check status:**  
   - Condition: `{{$json.status}} == "running"`  
   - Connect Snapshot Progress node output to If node input.  
   - True branch: Connect back to Wait node (loop).  
   - False branch: Proceed to next node.

6. **Add HTTP Request node to retrieve scraped data:**  
   - Type: HTTP Request (GET)  
   - URL: `https://api.brightdata.com/datasets/v3/snapshot/{{ $('HTTP Request- Post API call to Bright Data').item.json.snapshot_id }}`  
   - Query Parameter: format = json  
   - Headers: Authorization Bearer `<YOUR_BRIGHT_DATA_API_KEY>`  
   - Connect False branch of If node to this node.

7. **Add Google Sheets node to append reviews:**  
   - Type: Google Sheets (Append)  
   - Document ID: Use your copied Google Sheets template ID  
   - Sheet Name: `gid=0` (or your sheet tab)  
   - Columns: Auto-map input data fields (e.g., review_text, author_name, rating, etc.)  
   - Credentials: Configure Google Sheets OAuth2 credentials  
   - Connect HTTP Request (data retrieval) output to this node.

8. **Add Aggregate node to combine all reviews:**  
   - Type: Aggregate  
   - Aggregate field: `review_text`  
   - Output field name: `Aggregated_reviews`  
   - Connect Google Sheets append node output to this node.

9. **Add LangChain LLM Chain node for summarization:**  
   - Type: LangChain LLM Chain  
   - Prompt:  
     ```
     Read the following reviews, these are reviews of our competitors:
     {{ $json.Aggregated_reviews }}

     ---
     After reading them, summarize their weakest points.
     Don't mention the competitor name.
     ```  
   - Connect Aggregate node output to this node.

10. **Add OpenAI Chat Model node:**  
    - Type: OpenAI Chat Model  
    - Model: gpt-4o-mini  
    - Credentials: Configure OpenAI API key  
    - Connect LLM Chain node output to this node.

11. **Add LangChain OpenAI Image node to generate ad creative:**  
    - Type: LangChain OpenAI Image  
    - Prompt (JSON):  
      ```json
      {
        "ad_dimensions": {"width": 1080, "height": 1080},
        "target_audience": "B2C customer",
        "pain_points_source": "choose one pain point based on this {{ $json.text }}",
        "ad_requirements": {
          "image_style": "weird-and-fun",
          "weird_objects": ["Fruit with Faces", "Realistic People"],
          "focus": "address the biggest pain point of competitors",
          "avoid_naming_competitors": true,
          "headline": {"position": "No", "text_only": "No", "other_text": "No"},
          "background": ["bold red", "black"]
        }
      }
      ```  
    - Resource: image  
    - Credentials: OpenAI API key  
    - Connect OpenAI Chat Model output to this node.

12. **Add Gmail node to send email:**  
    - Type: Gmail (OAuth2)  
    - Recipient: your media buying team email (e.g., yaron.been@gmail.com)  
    - Subject: `Static Creatives Based on Our competitor {{ $now }}`  
    - Message:  
      ```
      Hey,

      We have analyzed our competitors recent reviews.
      Analysis data:
      {{ $today }}

      Here's a summary:
      {{ $('Basic LLM Chain - Summary of Recent reviews').item.json.text }}

      Please see attached an AI generated 1080x1080 image which you can use in Meta ads.
      ```  
    - Attachments: Attach binary image from OpenAI image generation node  
    - Credentials: Gmail OAuth2 credentials  
    - Connect OpenAI Image node output to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                   |
|---------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Workflow assistance and support contact: Yaron@nofluff.online                                                | Sticky Note9 content                                                                              |
| YouTube channel with tips and tutorials: https://www.youtube.com/@YaronBeen/videos                            | Sticky Note9 content                                                                              |
| LinkedIn profile: https://www.linkedin.com/in/yaronbeen/                                                     | Sticky Note9 content                                                                              |
| Bright Data documentation: https://docs.brightdata.com/introduction                                          | Sticky Note9 content                                                                              |
| Google Sheets template for storing reviews: https://docs.google.com/spreadsheets/d/1IR-yMQwEFTjbTCSPvVlQ54zQsnG0IRauTjPGoBWmR8U/edit?usp=sharing | Sticky Note10 content                                                                             |
| Prompt customization advice: Add company info and adjust summary requirements for better AI output            | Sticky Note4 content                                                                              |
| Bright Data scraping tips: update URLs frequently, allow more wait time for large scrapes, focus on targeted products | From workflow description                                                                         |
| Support email for help: Yaron@nofluff.online                                                                 | From workflow description                                                                         |

---

This documentation provides a detailed, structured reference for understanding, reproducing, and maintaining the "Scrape Competitor Reviews & Generate Ad Creatives with Bright Data & OpenAI" workflow. It covers all nodes, their configurations, connections, and potential failure points, enabling advanced users and AI agents to operate or extend the workflow confidently.
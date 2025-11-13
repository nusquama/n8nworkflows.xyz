Generate News Videos from RSS Feeds with HeyGen Avatar

https://n8nworkflows.xyz/workflows/generate-news-videos-from-rss-feeds-with-heygen-avatar-6229


# Generate News Videos from RSS Feeds with HeyGen Avatar

### 1. Workflow Overview

This workflow automates the generation of news videos using RSS feed data and the HeyGen avatar video creation API. It targets content creators or media teams who want to quickly transform news summaries into engaging video content featuring a digital avatar. The workflow is structured into three main logical blocks:

- **1.1 Trigger Input:** Manual trigger to initiate the process.
- **1.2 RSS Feed Parsing:** Reads and extracts news summaries from the Prothom Alo Bangla news RSS feed.
- **1.3 Video Generation:** Sends the extracted news summary to HeyGen API to generate avatar-based news videos with specific avatar and voice settings.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger Input

- **Overview:**  
  This block provides the manual starting point for the workflow, allowing users to execute it on demand.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - **Type & Role:** Manual Trigger node; initiates the workflow upon user action.  
    - **Configuration:** No parameters; triggers when the “Execute workflow” button is pressed.  
    - **Expressions/Variables:** None used here.  
    - **Input/Output:** No inputs; outputs a trigger event to the next node.  
    - **Version Requirements:** Compatible with n8n version ≥1.0.  
    - **Edge Cases / Failures:** None expected; manual trigger relies on user action.  
    - **Sub-workflow:** None.

#### 2.2 RSS Feed Parsing

- **Overview:**  
  This block reads the latest news articles from the Prothom Alo Bangla RSS feed, providing structured data including summaries used for video content.

- **Nodes Involved:**  
  - Read RSS Feed from Prothom Alo

- **Node Details:**

  - **Read RSS Feed from Prothom Alo**  
    - **Type & Role:** RSS Feed Read node; fetches and parses RSS XML data from the specified URL.  
    - **Configuration:**  
      - URL: `https://prod-qt-images.s3.amazonaws.com/production/prothomalo-bangla/feed.xml`  
      - Default options; no filters or limits set.  
    - **Expressions/Variables:** None directly used; outputs JSON with feed items including `summary`.  
    - **Input/Output:** Receives trigger from manual node; outputs parsed RSS items to the next node.  
    - **Version Requirements:** Compatible with n8n version ≥1.2 recommended for stable RSS parsing.  
    - **Edge Cases / Failures:**  
      - Network issues or RSS feed unavailability.  
      - Malformed RSS XML can cause parsing errors.  
      - Feed structure changes may affect summary extraction.  
    - **Sub-workflow:** None.

#### 2.3 Video Generation

- **Overview:**  
  This block transforms the news summary into an AI-generated video featuring a HeyGen avatar with specified voice and visual settings.

- **Nodes Involved:**  
  - Generate Video News

- **Node Details:**

  - **Generate Video News**  
    - **Type & Role:** HTTP Request node; sends a POST request to HeyGen’s video generation API.  
    - **Configuration:**  
      - Method: POST  
      - URL: `https://api.heygen.com/v2/video/generate`  
      - Headers:  
        - `X-Api-Key`: API key (must be configured securely in credentials or environment variables)  
        - `Content-Type`: `application/json`  
      - Body (JSON):  
        ```json
        {
          "video_inputs": [
            {
              "character": {
                "type": "avatar",
                "avatar_id": "Lina_Dress_Sitting_Side_public",
                "avatar_style": "normal"
              },
              "voice": {
                "type": "text",
                "input_text": "{{ $json.summary }}",
                "voice_id": "119caed25533477ba63822d5d1552d25",
                "speed": 1.1
              }
            }
          ],
          "dimension": {
            "width": 1280,
            "height": 720
          }
        }
        ```  
      - The `input_text` field dynamically references the `summary` field from the RSS feed JSON.  
    - **Expressions/Variables:** `{{ $json.summary }}` — injects the news summary text.  
    - **Input/Output:** Receives parsed RSS feed items; outputs API response (video generation details).  
    - **Version Requirements:** HTTP Request node v4.2 or higher recommended for enhanced JSON body support.  
    - **Edge Cases / Failures:**  
      - API key invalid or expired (authentication failure).  
      - Network timeouts or API service downtime.  
      - Empty or malformed summary text causing video generation errors.  
      - Rate limiting or quota exceeded on HeyGen API.  
    - **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name                   | Node Type           | Functional Role               | Input Node(s)              | Output Node(s)           | Sticky Note                                                                                           |
|-----------------------------|---------------------|------------------------------|----------------------------|--------------------------|-----------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger      | Start workflow manually        | —                          | Read RSS Feed from Prothom Alo |                                                                                                     |
| Read RSS Feed from Prothom Alo    | RSS Feed Read       | Fetch and parse RSS feed       | When clicking ‘Execute workflow’ | Generate Video News        |                                                                                                     |
| Generate Video News           | HTTP Request        | Call HeyGen API to generate video | Read RSS Feed from Prothom Alo  | —                        | ## Generate News Videos from RSS Feeds with HeyGen Avatar<br>**Steps:**<br>- Triggered manually via the "Execute workflow" button.<br>- Reads RSS feed from Prothom Alo Bangla news XML.<br>- Uses news summary from the feed as input text for video generation.<br>- Sends a POST request to HeyGen API to create videos with specified avatar and voice settings.<br>- API key is included in the HTTP request headers (ensure it is kept secure).<br>- Video dimension is set to 1280x720 pixels. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Node Type: Manual Trigger  
   - Name: `When clicking ‘Execute workflow’`  
   - No parameters needed.

2. **Create an RSS Feed Read Node**  
   - Node Type: RSS Feed Read  
   - Name: `Read RSS Feed from Prothom Alo`  
   - Parameters:  
     - URL: `https://prod-qt-images.s3.amazonaws.com/production/prothomalo-bangla/feed.xml`  
   - Connect the output of the Manual Trigger node to this node’s input.

3. **Create an HTTP Request Node**  
   - Node Type: HTTP Request  
   - Name: `Generate Video News`  
   - Parameters:  
     - HTTP Method: POST  
     - URL: `https://api.heygen.com/v2/video/generate`  
     - Authentication: None (API key passed in header)  
     - Headers:  
       - `X-Api-Key`: (Set your HeyGen API key securely here)  
       - `Content-Type`: `application/json`  
     - Body Content Type: JSON  
     - Body JSON:  
       ```json
        {
          "video_inputs": [
            {
              "character": {
                "type": "avatar",
                "avatar_id": "Lina_Dress_Sitting_Side_public",
                "avatar_style": "normal"
              },
              "voice": {
                "type": "text",
                "input_text": "{{ $json.summary }}",
                "voice_id": "119caed25533477ba63822d5d1552d25",
                "speed": 1.1
              }
            }
          ],
          "dimension": {
            "width": 1280,
            "height": 720
          }
        }
       ```  
   - Connect the output of the RSS Feed Read node to this HTTP Request node’s input.

4. **Configure Credentials**  
   - For the HTTP Request node, ensure to store and use your HeyGen API key securely. This can be done via n8n credentials or environment variables.

5. **Test the Workflow**  
   - Manually trigger the workflow.  
   - Verify that the RSS feed is read correctly and news summaries are extracted.  
   - Confirm the POST request is sent to the HeyGen API with valid payload and headers.  
   - Check the API response for video generation status.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                      | Context or Link                                                                                   |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| The workflow uses a HeyGen avatar named "Lina_Dress_Sitting_Side_public" with "normal" avatar style and a specific voice ID for voice synthesis. The video resolution is set to 1280x720 (HD) for standard viewing quality.       | HeyGen API documentation (not linked in workflow)                                              |
| API key security is critical; never expose the key in raw workflow JSON or public repositories. Use n8n’s credential management to store sensitive keys.                                                                       | n8n Credentials documentation: https://docs.n8n.io/credentials/                                  |
| RSS feed URL is from Prothom Alo Bangla news, which may change or be unavailable; consider adding error handling or fallback feeds for production use.                                                                          | RSS feed standards: https://validator.w3.org/feed/docs/rss2.html                                |
| Manual triggering is suitable for testing or low-frequency runs. For automation, consider using Scheduled Trigger nodes or webhook triggers depending on use case.                                                             | n8n Trigger nodes documentation: https://docs.n8n.io/nodes/n8n-nodes-base.manualTrigger/         |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.
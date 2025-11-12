Auto-Generate YouTube Shorts from Reddit Content with OpenAI Using MCP Server

https://n8nworkflows.xyz/workflows/auto-generate-youtube-shorts-from-reddit-content-with-openai-using-mcp-server-4018


# Auto-Generate YouTube Shorts from Reddit Content with OpenAI Using MCP Server

---
### 1. Workflow Overview

This workflow automates the creation of YouTube Shorts videos by leveraging Reddit content and AI-powered tools integrated through a custom MCP Server. It targets content creators, marketers, and business owners who want to generate engaging short videos automatically without manual editing or complex video production skills.

The workflow logically splits into three parallel but structurally similar pipelines (likely for different Reddit sources or content batches). Each pipeline consists of these functional blocks:

- **1.1 Input Reception:** Fetch top weekly Reddit posts via RSS feeds.
- **1.2 Data Mapping and Aggregation:** Prepare and aggregate content data for AI processing.
- **1.3 AI Script & Video Generation:** Use OpenAI to generate video scripts, then invoke the MCP Client tool to trigger video assembly.
- **1.4 Video Status Monitoring:** Wait for video generation completion, check status, and decide whether to download.
- **1.5 Video Download & Publishing:** Download the completed video and optionally share it on YouTube.

These blocks are repeated three times with identical node structures, possibly to support multiple content streams or topics simultaneously.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Reads top weekly posts from Reddit using RSS feeds to serve as raw content inputs.

- **Nodes Involved:**  
  - Get top weekly posts (three instances)

- **Node Details:**

  - **Get top weekly posts (and its duplicates):**  
    - Type: RSS Feed Read  
    - Role: Fetches Reddit top weekly posts as input content.  
    - Configuration: Uses RSS feed URLs pointing to Reddit's top weekly posts.  
    - Inputs: Triggered by manual trigger node or prior nodes.  
    - Outputs: Emits an array of posts for further processing.  
    - Edge cases: RSS feed unavailability, malformed feed, or rate limiting.  
    - Notes: Critical starting point; failure here halts downstream processing.

#### 1.2 Data Mapping and Aggregation

- **Overview:**  
  Prepares and aggregates the fetched Reddit posts into a format suitable for AI processing.

- **Nodes Involved:**  
  - Map fields (three instances)  
  - Aggregate (three instances)

- **Node Details:**

  - **Map fields:**  
    - Type: Set  
    - Role: Extracts and normalizes relevant fields from RSS items (e.g., post titles, URLs).  
    - Configuration: Maps specific RSS feed attributes to new structured fields.  
    - Inputs: RSS feed outputs.  
    - Outputs: Structured data objects for aggregation.  
    - Edge cases: Missing expected fields, null values.

  - **Aggregate:**  
    - Type: Aggregate  
    - Role: Combines multiple post items into a single data structure to batch for AI input.  
    - Configuration: Default aggregation, likely concatenating or grouping items.  
    - Inputs: Outputs from Map fields node.  
    - Outputs: Aggregated array/object for AI consumption.  
    - Edge cases: Large data volumes causing timeout or memory issues.

#### 1.3 AI Script & Video Generation

- **Overview:**  
  Generates SEO-optimized video scripts from aggregated posts using OpenAI, then initiates video generation via MCP Client.

- **Nodes Involved:**  
  - OpenAI Chat Model (three instances)  
  - Structured Output Parser (three instances)  
  - MCP Client (three instances)  
  - Generate video (agent node, three instances)

- **Node Details:**

  - **OpenAI Chat Model:**  
    - Type: Language Model Chat (OpenAI)  
    - Role: Produces engaging, scripted content for YouTube Shorts based on input data.  
    - Configuration: Uses OpenAI API credentials, prompts possibly tailored for SEO and engagement.  
    - Inputs: Aggregated Reddit posts.  
    - Outputs: Raw AI-generated script text or structured response.  
    - Edge cases: API rate limits, malformed prompts, unexpected AI outputs.

  - **Structured Output Parser:**  
    - Type: Langchain structured output parser  
    - Role: Parses OpenAI output into a predefined structured format for downstream processing.  
    - Configuration: Output schema defined to extract script and metadata cleanly.  
    - Inputs: Raw AI output.  
    - Outputs: Structured data objects with script and video parameters.  
    - Edge cases: Parsing failures due to unexpected AI format changes.

  - **MCP Client:**  
    - Type: MCP Server Client Tool  
    - Role: Sends structured script and media instructions to MCP Server to trigger video assembly.  
    - Configuration: Connected to custom MCP Server with appropriate credentials.  
    - Inputs: Parsed structured script data.  
    - Outputs: Video generation job initiation responses.  
    - Edge cases: Network errors, authentication errors, server timeouts.

  - **Generate video:**  
    - Type: Langchain agent node  
    - Role: Orchestrates the AI tool (OpenAI + MCP Client) chain to execute the video generation logic.  
    - Inputs: Linked from MCP Client and OpenAI Chat Model nodes.  
    - Outputs: Job data used to monitor video generation status.  
    - Edge cases: Agent timeout or tool coordination failures.

#### 1.4 Video Status Monitoring

- **Overview:**  
  Monitors video generation status by polling MCP Server until completion or failure.

- **Nodes Involved:**  
  - Wait (three instances)  
  - Check video status (HTTP Request, three instances)  
  - If (three instances)

- **Node Details:**

  - **Wait:**  
    - Type: Wait  
    - Role: Pauses workflow execution for a set interval before rechecking video status to avoid excessive polling.  
    - Configuration: Fixed delay or webhook-based waiting.  
    - Inputs: Post video generation job creation.  
    - Outputs: Triggers status check after wait.  
    - Edge cases: Overly long wait causing delays, premature triggers.

  - **Check video status:**  
    - Type: HTTP Request  
    - Role: Queries MCP Server API to get current status of video rendering job.  
    - Configuration: Uses job ID, authentication headers, and MCP API endpoint.  
    - Inputs: Wait node output.  
    - Outputs: Job status JSON.  
    - Edge cases: API failure, network issues, unauthorized access.

  - **If:**  
    - Type: Conditional branch  
    - Role: Checks if video is completed or still processing; routes flow accordingly.  
    - Configuration: Condition based on status field in API response.  
    - Inputs: Check video status output.  
    - Outputs:  
      - True branch: Video ready to download.  
      - False branch: Loop back to Wait node for continued polling.  
    - Edge cases: Incorrect status parsing leading to infinite loops or premature termination.

#### 1.5 Video Download & Publishing

- **Overview:**  
  Downloads the completed video file and optionally publishes it to YouTube.

- **Nodes Involved:**  
  - Download the video (HTTP Request, three instances)  
  - Share on YouTube (YouTube node, three instances)

- **Node Details:**

  - **Download the video:**  
    - Type: HTTP Request  
    - Role: Fetches the rendered video file from MCP Server or CDN storage.  
    - Configuration: Uses video URL from status response, handles file streaming.  
    - Inputs: If node's true branch upon video completion.  
    - Outputs: Video binary data for next steps.  
    - Edge cases: Download failures, large file handling issues.

  - **Share on YouTube:**  
    - Type: YouTube  
    - Role: Uploads the downloaded video to a YouTube channel.  
    - Configuration: OAuth2 credentials linked to YouTube account, metadata such as title, description, tags.  
    - Inputs: Video binary from download node.  
    - Outputs: Upload confirmation and video URL.  
    - Edge cases: Authentication expiry, quota limits, upload failures.

---

### 3. Summary Table

| Node Name                 | Node Type                                     | Functional Role                           | Input Node(s)                      | Output Node(s)                     | Sticky Note                                  |
|---------------------------|-----------------------------------------------|-----------------------------------------|----------------------------------|----------------------------------|----------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                              | Entry point to start workflow           | -                                | Get top weekly posts              |                                              |
| Get top weekly posts       | RSS Feed Read                                | Fetch Reddit top weekly posts            | When clicking ‘Test workflow’     | Map fields                       |                                              |
| Map fields                | Set                                           | Prepare and map RSS feed data            | Get top weekly posts              | Aggregate                       |                                              |
| Aggregate                 | Aggregate                                     | Aggregate posts for AI input             | Map fields                       | Generate video                   |                                              |
| OpenAI Chat Model          | Langchain OpenAI Chat Model                   | Generate video script from aggregated data | Aggregate                       | Structured Output Parser          |                                              |
| Structured Output Parser   | Langchain Output Parser Structured             | Parse AI script output to structured format | OpenAI Chat Model               | MCP Client                      |                                              |
| MCP Client                | Langchain MCP Client Tool                      | Send script to MCP Server for video generation | Structured Output Parser         | Generate video (agent)            |                                              |
| Generate video             | Langchain Agent                               | Orchestrates AI tools and triggers video generation | MCP Client, OpenAI Chat Model     | Wait                             |                                              |
| Wait                      | Wait                                          | Wait interval before checking video status | Generate video                  | Check video status                |                                              |
| Check video status         | HTTP Request                                  | Poll MCP Server for video generation status | Wait                          | If                              |                                              |
| If                        | If Node                                       | Conditional branch for video ready or wait | Check video status              | Download the video / Wait        |                                              |
| Download the video         | HTTP Request                                  | Download completed video file            | If (true branch)                 | Share on YouTube                 |                                              |
| Share on YouTube           | YouTube                                       | Upload video to YouTube channel          | Download the video               | -                               |                                              |
| Get top weekly posts1      | RSS Feed Read                                | Parallel RSS input for second content stream | When clicking ‘Test workflow’   | Map fields1                     |                                              |
| Map fields1               | Set                                           | Map fields for second stream              | Get top weekly posts1            | Aggregate1                      |                                              |
| Aggregate1                | Aggregate                                     | Aggregate for second stream               | Map fields1                     | Generate video1                 |                                              |
| OpenAI Chat Model1         | Langchain OpenAI Chat Model                   | Generate script for second stream         | Aggregate1                     | Structured Output Parser1        |                                              |
| Structured Output Parser1  | Langchain Output Parser Structured             | Parse AI output for second stream         | OpenAI Chat Model1             | MCP Client1                    |                                              |
| MCP Client1               | Langchain MCP Client Tool                      | Send to MCP Server for second stream      | Structured Output Parser1        | Generate video1 (agent)          |                                              |
| Generate video1            | Langchain Agent                               | Orchestrate second stream video generation | MCP Client1, OpenAI Chat Model1  | Wait1                          |                                              |
| Wait1                     | Wait                                          | Wait for second stream video generation  | Generate video1                 | Check video status1              |                                              |
| Check video status1        | HTTP Request                                  | Poll status for second stream             | Wait1                          | If1                            |                                              |
| If1                       | If Node                                       | Conditional branch for second stream      | Check video status1             | Download the video1 / Wait1      |                                              |
| Download the video1        | HTTP Request                                  | Download video for second stream          | If1 (true branch)               | Share on YouTube1               |                                              |
| Share on YouTube1          | YouTube                                       | Upload second stream video                 | Download the video1             | -                               |                                              |
| Get top weekly posts2      | RSS Feed Read                                | Third parallel RSS input                   | When clicking ‘Test workflow’   | Map fields2                    |                                              |
| Map fields2               | Set                                           | Map fields for third stream                 | Get top weekly posts2           | Aggregate2                     |                                              |
| Aggregate2                | Aggregate                                     | Aggregate for third stream                  | Map fields2                    | Generate video2                |                                              |
| OpenAI Chat Model2         | Langchain OpenAI Chat Model                   | Generate script for third stream            | Aggregate2                    | Structured Output Parser2       |                                              |
| Structured Output Parser2  | Langchain Output Parser Structured             | Parse AI output for third stream            | OpenAI Chat Model2            | MCP Client2                   |                                              |
| MCP Client2               | Langchain MCP Client Tool                      | Send to MCP Server third stream             | Structured Output Parser2       | Generate video2 (agent)         |                                              |
| Generate video2            | Langchain Agent                               | Orchestrate third stream video generation   | MCP Client2, OpenAI Chat Model2 | Wait2                         |                                              |
| Wait2                     | Wait                                          | Wait for third stream video generation     | Generate video2                | Check video status2             |                                              |
| Check video status2        | HTTP Request                                  | Poll status for third stream                 | Wait2                         | If2                           |                                              |
| If2                       | If Node                                       | Conditional branch for third stream          | Check video status2            | Download the video2 / Wait2      |                                              |
| Download the video2        | HTTP Request                                  | Download video for third stream              | If2 (true branch)              | Share on YouTube2              |                                              |
| Share on YouTube2          | YouTube                                       | Upload third stream video                     | Download the video2            | -                               |                                              |
| When clicking ‘Test workflow’ | Manual Trigger                              | Entry point for workflow initiation          | -                              | Get top weekly posts, etc.       |                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Type: Manual Trigger  
   - Purpose: Entry point to start the entire workflow.

2. **Create RSS Feed Read Node (Get top weekly posts):**  
   - Type: RSS Feed Read  
   - Configure URL to Reddit’s top weekly posts RSS feed (e.g., https://www.reddit.com/r/subreddit/top/.rss?t=week).  
   - Connect output to Map fields node.  
   - Duplicate this node two more times for parallel streams if needed.

3. **Create Set Node (Map fields):**  
   - Type: Set  
   - Configure to extract relevant fields such as title, link, description from RSS feed items.  
   - Connect input from respective RSS nodes.  
   - Connect output to Aggregate node.

4. **Create Aggregate Node:**  
   - Type: Aggregate  
   - Configure default aggregation to batch the mapped posts into an array/object.  
   - Connect input from Map fields node.  
   - Connect output to OpenAI Chat Model node.

5. **Create OpenAI Chat Model Node:**  
   - Type: Langchain OpenAI Chat Model  
   - Configure with OpenAI API credentials.  
   - Use prompts designed to generate YouTube Shorts scripts based on aggregated Reddit posts, focusing on engagement and SEO.  
   - Connect input from Aggregate node.  
   - Connect output to Structured Output Parser node.

6. **Create Structured Output Parser Node:**  
   - Type: Langchain Structured Output Parser  
   - Define schema to parse AI response into a structured format (script text, video parameters).  
   - Connect input from OpenAI Chat Model node.  
   - Connect output to MCP Client node.

7. **Create MCP Client Node:**  
   - Type: Langchain MCP Client Tool  
   - Configure connection details and credentials for your custom MCP Server.  
   - Input structured script data.  
   - Connect output to Generate video node.

8. **Create Generate Video Node:**  
   - Type: Langchain Agent  
   - Connect inputs from MCP Client and OpenAI Chat Model nodes as AI tools.  
   - This node orchestrates the generation.  
   - Connect output to Wait node.

9. **Create Wait Node:**  
   - Type: Wait  
   - Configure delay interval to avoid excessive polling (e.g., 30 seconds).  
   - Connect input from Generate video node.  
   - Connect output to Check video status node.

10. **Create Check Video Status Node:**  
    - Type: HTTP Request  
    - Configure to call MCP Server API with the video job ID to check status.  
    - Use authentication headers as required.  
    - Connect input from Wait node.  
    - Connect output to If node.

11. **Create If Node:**  
    - Type: If  
    - Configure condition to check if video status equals “completed” or relevant success status.  
    - True branch connects to Download the video node.  
    - False branch loops back to Wait node for re-polling.

12. **Create Download the Video Node:**  
    - Type: HTTP Request  
    - Configure to download video file from MCP Server URL.  
    - Connect input from If node (true branch).  
    - Connect output to Share on YouTube node.

13. **Create Share on YouTube Node:**  
    - Type: YouTube  
    - Configure OAuth2 credentials for YouTube account.  
    - Set video metadata (title, description, tags).  
    - Connect input from Download the video node.

14. **Replicate Steps 2-13 for each parallel content stream:**  
    - Duplicate all nodes with suffixes (e.g., Get top weekly posts1, Map fields1, etc.)  
    - Connect each chain individually from the manual trigger node.

15. **Test Workflow:**  
    - Trigger the manual trigger node.  
    - Monitor logs and outputs for each stage.  
    - Ensure video generation completes and videos are uploaded to YouTube.

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                                                          |
|-----------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------|
| Workflow uses a custom MCP Server to automate video assembly from AI-generated scripts.        | Custom MCP Server integration critical for video creation.                              |
| Requires OpenAI API credentials for script generation.                                        | Obtain API key from https://openai.com/api                                            |
| YouTube OAuth2 credentials needed for video upload automation.                                | Setup via Google Cloud Console: https://console.cloud.google.com/apis/credentials      |
| Reddit RSS feeds provide trending content inputs for script generation.                       | Example Reddit RSS: https://www.reddit.com/r/all/top/.rss?t=week                        |
| Video generation status polling uses HTTP requests and wait nodes to handle asynchronous jobs. | Avoid setting too short wait intervals to prevent rate limiting or API overload.       |
| Workflow supports customization of script tone and video style via AI prompt adjustment.      | Adjust OpenAI prompt templates for branding or tone modifications.                      |
| MCP Server handles video assembly, voiceover, and editing steps automatically.                | Ensure MCP Server endpoints and credentials are correctly configured.                   |

---

**Disclaimer:**  
The provided information is extracted exclusively from an n8n automated workflow. It complies strictly with content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.
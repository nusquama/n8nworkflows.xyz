Generate Text & Image Responses in Telegram Channels with GPT-4 and TGPT

https://n8nworkflows.xyz/workflows/generate-text---image-responses-in-telegram-channels-with-gpt-4-and-tgpt-9818


# Generate Text & Image Responses in Telegram Channels with GPT-4 and TGPT

### 1. Workflow Overview

This workflow implements an automated Telegram AI Channel Bot that monitors a specified Telegram channel for new messages and generates AI-driven responses using the GPT-4 model via the TGPT tool. It supports two types of AI-generated content based on message prefixes: text responses and image generation. The bot polls the Telegram API every 10 seconds, processes new channel posts while avoiding duplicates, and responds accordingly.

Logical blocks within the workflow:

- **1.1 Scheduling & Configuration**: Periodically triggers the workflow and initializes configuration parameters including bot token, channel ID, and message offset tracking.
- **1.2 Telegram Updates Polling & Offset Processing**: Fetches new channel posts from Telegram, manages offset and duplicate prevention, and filters messages within a recent time window.
- **1.3 Message Filtering & Routing**: Validates messages originate from the configured channel and routes them based on command prefixes (`am#` for text, `ami#` for images).
- **1.4 Text Generation Pipeline**: Processes text generation requests by invoking TGPT with GPT-4, cleaning the output, and sending the text response back to the Telegram channel.
- **1.5 Image Generation Pipeline**: Processes image generation requests by invoking TGPT to generate an image file, reading the generated image, and sending it as a photo message to the Telegram channel.
- **1.6 Offset Cleanup**: Ensures the Telegram update offset is updated to avoid reprocessing the same messages.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduling & Configuration

**Overview:**  
This block triggers the workflow every 10 seconds and sets the initial configuration parameters required for Telegram API interactions and message offset tracking.

**Nodes Involved:**  
- Schedule  
- Config

**Node Details:**

- **Schedule**  
  - Type: Schedule Trigger  
  - Configuration: Runs every 10 seconds (seconds interval = 10).  
  - Input: None (trigger node)  
  - Output: Triggers next node to start workflow cycle.  
  - Edge Cases: None typical; ensure server time is accurate to maintain consistent polling.

- **Config**  
  - Type: Set  
  - Configuration: Sets three variables:  
    - `bot_token` (string): Telegram bot token (to be replaced by user).  
    - `channel_id` (string): Telegram channel ID monitored by bot.  
    - `last_offset` (number): Tracks last processed Telegram update offset, initialized from global static data or zero.  
  - Input: Triggered by Schedule node.  
  - Output: Provides configuration data to downstream nodes.  
  - Edge Cases: User must replace placeholder tokens; incorrect tokens or channel IDs will cause API failures.

---

#### 1.2 Telegram Updates Polling & Offset Processing

**Overview:**  
Fetches new channel posts via Telegram API `getUpdates` method, applies offset tracking to prevent duplicate processing, and filters messages to only process recent ones within a 15-second window.

**Nodes Involved:**  
- Get Updates  
- Process Offset  
- Clear Update List

**Node Details:**

- **Get Updates**  
  - Type: HTTP Request  
  - Configuration:  
    - URL: Telegram API `getUpdates` endpoint using bot token from Config node.  
    - Query Parameters:  
      - `allowed_updates`: Only `channel_post` updates.  
      - `timeout`: 3 seconds long polling.  
      - `offset`: last processed offset from Config.  
      - `limit`: 15 updates max per call.  
    - Timeout: 30 seconds.  
  - Input: Receives Config data with bot token and last_offset.  
  - Output: JSON containing Telegram updates.  
  - Failure Modes: Network timeout, invalid token, or API rate limits.

- **Process Offset**  
  - Type: Code (JavaScript)  
  - Configuration:  
    - Processes received updates to determine which messages are new and recent (within 15 seconds).  
    - Maintains a global static list of processed update IDs to prevent duplicates.  
    - Updates last_offset to highest update_id + 1.  
    - Logs debug info for message filtering.  
    - Returns only new messages with offset info or a placeholder if none.  
  - Input: JSON updates from Get Updates node.  
  - Output: Filtered new messages or offset info for downstream processing.  
  - Edge Cases: Handles empty updates gracefully, avoids processing old or duplicate messages.  
  - Possible failures: Expression errors in code, static data corruption.

- **Clear Update List**  
  - Type: HTTP Request  
  - Configuration: Calls `getUpdates` with current offset to clear Telegram update queue.  
  - Input: Offset data from Process Offset node.  
  - Output: Generally discarded; used to clear API queue.  
  - Failure Modes: Same as Get Updates node; continues on failure.

---

#### 1.3 Message Filtering & Routing

**Overview:**  
Filters messages to ensure they originate from the configured Telegram channel and routes them based on message prefix commands (`am#` for text, `ami#` for images).

**Nodes Involved:**  
- Split  
- Filter  
- Switch

**Node Details:**

- **Split**  
  - Type: Split Out  
  - Configuration: Splits input items by the field `channel_post.text` to process each message text individually.  
  - Input: Array of new messages from Process Offset node.  
  - Output: Individual message text items.  
  - Edge Cases: Messages without text are ignored or filtered out downstream.

- **Filter**  
  - Type: If  
  - Configuration: Checks if `channel_post.chat.id` matches configured `channel_id`.  
  - Input: Each message item from Split node.  
  - Output: Passes only messages from the target channel.  
  - Failure Modes: Missing fields cause filtering to fail; strict type checking applied.

- **Switch**  
  - Type: Switch  
  - Configuration: Routes messages based on prefix in `channel_post.text`:  
    - `am# ` prefix routes to text generation flow.  
    - `ami# ` prefix routes to image generation flow.  
  - Input: Filtered messages from Filter node.  
  - Output: Branch 1 to Text, Branch 2 to Image nodes.  
  - Edge Cases: Messages without recognized prefix are discarded.

---

#### 1.4 Text Generation Pipeline

**Overview:**  
Generates AI text responses using GPT-4 through TGPT CLI tool, cleans the output, and sends the generated text back to the Telegram channel.

**Nodes Involved:**  
- Execute - Text  
- Clean  
- Send Telegram Text Response

**Node Details:**

- **Execute - Text**  
  - Type: Execute Command  
  - Configuration:  
    - Installs necessary packages (`util-linux-misc`, `curl`) if missing.  
    - Downloads TGPT binary from GitHub release.  
    - Removes `am#` prefix from message text to form prompt.  
    - Runs TGPT CLI with GPT-4 model, temperature 0.3 for focused text generation.  
    - Captures output to temporary file `/tmp/response.txt`.  
    - Filters output to remove web search lines.  
  - Input: Messages routed from Switch node with `am#` prefix.  
  - Output: Raw AI-generated text output.  
  - Failure Modes: CLI failures, download errors, command execution errors.  
  - Continue on Fail enabled to prevent workflow halt.

- **Clean**  
  - Type: Code (JavaScript)  
  - Configuration:  
    - Checks if `stdout` output exists.  
    - Cleans and prepares human-readable text from raw output.  
    - Returns cleaned text or error message if no output.  
  - Input: Raw output from Execute - Text node.  
  - Output: Cleaned text ready for Telegram.  
  - Edge Cases: Missing or empty output handled gracefully.

- **Send Telegram Text Response**  
  - Type: Telegram  
  - Configuration:  
    - Sends cleaned text to configured Telegram channel.  
    - Uses HTML parse mode with no attribution appended.  
    - Uses Telegram bot credentials (configured with token).  
  - Input: Cleaned text from Clean node.  
  - Output: Telegram message sent confirmation.  
  - Failure Modes: Invalid credentials, network issues, Telegram API errors.

---

#### 1.5 Image Generation Pipeline

**Overview:**  
Generates images using GPT-4 through TGPT CLI, reads the generated image file, and sends the image as a photo to the Telegram channel.

**Nodes Involved:**  
- Execute - Image  
- Read Generated Image  
- Send Telegram Image Response

**Node Details:**

- **Execute - Image**  
  - Type: Execute Command  
  - Configuration:  
    - Installs `util-linux-misc` if missing.  
    - Downloads TGPT binary.  
    - Removes `ami#` prefix from message text to form prompt.  
    - Runs TGPT CLI with image generation parameters: 1920x1080 resolution, temperature 0.7 for creativity.  
    - Outputs image to `/tmp/genimg.jpg`.  
  - Input: Messages routed from Switch node with `ami#` prefix.  
  - Output: Generates image file saved locally.  
  - Failure Modes: CLI failures, missing dependencies, file write errors.  
  - Continue on Fail enabled.

- **Read Generated Image**  
  - Type: Read/Write File  
  - Configuration: Reads image file `/tmp/genimg.jpg` into a binary property `genimg`.  
  - Input: Triggered after Execute - Image node completes.  
  - Output: Binary data of generated image for sending.  
  - Failure Modes: File not found, permission errors.

- **Send Telegram Image Response**  
  - Type: Telegram  
  - Configuration:  
    - Sends binary image data as a photo to the configured Telegram channel.  
    - Uses same Telegram API credentials as text response node.  
  - Input: Binary image data from Read Generated Image node.  
  - Output: Confirmation of photo sent to channel.  
  - Failure Modes: Invalid credentials, Telegram API errors, network issues.

---

#### 1.6 Offset Cleanup

**Overview:**  
Ensures Telegram update offset is updated after processing messages to avoid reprocessing.

**Nodes Involved:**  
- Clear Update List (already described in 1.2)

---

### 3. Summary Table

| Node Name                | Node Type             | Functional Role                             | Input Node(s)           | Output Node(s)                | Sticky Note                                                                                                          |
|--------------------------|-----------------------|---------------------------------------------|-------------------------|------------------------------|----------------------------------------------------------------------------------------------------------------------|
| Schedule                 | Schedule Trigger      | Triggers workflow every 10 seconds           | None                    | Config                       | ## Telegram AI Channel Bot Automates AI response generation, polling every 10 seconds with duplicate prevention.     |
| Config                   | Set                   | Sets bot token, channel ID, last message offset | Schedule               | Get Updates                  | ## CONFIGURATION REQUIRED Bot token and channel ID must be replaced; bot must be admin in the channel. Offset auto. |
| Get Updates              | HTTP Request          | Fetches new channel posts from Telegram API  | Config                   | Process Offset               | ## Message Polling & Processing Fetches messages, applies time window and duplicate filtering.                       |
| Process Offset           | Code (JavaScript)     | Filters new messages, updates offset          | Get Updates              | Split, Clear Update List     | ## Message Polling & Processing, see above.                                                                           |
| Clear Update List        | HTTP Request          | Clears Telegram update queue                   | Process Offset           | None                        | ## Message Polling & Processing, see above.                                                                           |
| Split                    | Split Out             | Splits messages array into individual texts   | Process Offset           | Filter                      | ## Filtering & Message Routing Filters and processes individual texts.                                               |
| Filter                   | If                    | Validates message channel origin                | Split                    | Switch                      | ## Filtering & Message Routing Validates messages come from configured channel.                                      |
| Switch                   | Switch                | Routes messages based on prefix (`am#`, `ami#`) | Filter                  | Execute - Text, Execute - Image | ## Filtering & Message Routing Routes messages to text or image generation pipelines.                                  |
| Execute - Text           | Execute Command       | Runs TGPT to generate text response             | Switch (am)              | Clean                       | ## Text Generation Pipeline Installs TGPT, generates GPT-4 text with temperature 0.3, cleans output.                 |
| Clean                    | Code (JavaScript)     | Cleans raw TGPT output for text                  | Execute - Text           | Send Telegram Text Response | ## Text Generation Pipeline Cleans AI output, handles empty output gracefully.                                        |
| Send Telegram Text Response | Telegram           | Sends AI-generated text to Telegram channel      | Clean                    | None                        | ## TELEGRAM CREDENTIALS Uses Telegram credentials; sends text with HTML parsing.                                      |
| Execute - Image          | Execute Command       | Runs TGPT to generate image                       | Switch (ami)             | Read Generated Image        | ## Image Generation Pipeline Installs TGPT, generates 1920x1080 image with temperature 0.7.                           |
| Read Generated Image     | Read/Write File       | Reads generated image file into binary property  | Execute - Image          | Send Telegram Image Response | ## Image Generation Pipeline Reads image from /tmp/ for sending.                                                     |
| Send Telegram Image Response | Telegram           | Sends AI-generated image to Telegram channel     | Read Generated Image     | None                        | ## TELEGRAM CREDENTIALS Uses Telegram credentials; sends image as photo message.                                      |
| Sticky Note              | Sticky Note           | Documentation and notes                          | None                    | None                        | See individual sticky note contents in other rows.                                                                   |
| Sticky Note1             | Sticky Note           | Configuration instructions                       | None                    | None                        | See Config node.                                                                                                      |
| Sticky Note2             | Sticky Note           | Polling and offset processing explanation        | None                    | None                        | See Get Updates and Process Offset nodes.                                                                             |
| Sticky Note3             | Sticky Note           | Filtering and routing explanation                 | None                    | None                        | See Split, Filter, Switch nodes.                                                                                      |
| Sticky Note4             | Sticky Note           | Text generation pipeline explanation              | None                    | None                        | See Execute - Text and Clean nodes.                                                                                   |
| Sticky Note5             | Sticky Note           | Image generation pipeline explanation             | None                    | None                        | See Execute - Image and Read Generated Image nodes.                                                                   |
| Sticky Note6             | Sticky Note           | Telegram credentials instructions                  | None                    | None                        | See Send Telegram Text Response and Send Telegram Image Response nodes.                                               |
| Sticky Note7             | Sticky Note           | Usage command examples                             | None                    | None                        | Shows example commands: `am# Explain quantum physics`, `ami# Futuristic city at sunset`.                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Set to trigger every 10 seconds (`secondsInterval: 10`).  
   - This node triggers the entire workflow cycle.

2. **Create Config Set Node**  
   - Type: Set  
   - Add three variables:  
     - `bot_token` (string): Your Telegram bot token from @BotFather.  
     - `channel_id` (string): Your Telegram channel ID.  
     - `last_offset` (number): Use expression to read from global static data or default to 0: `{{$getWorkflowStaticData('global').last_offset || 0}}`.  
   - Connect Schedule node output to this node input.

3. **Create HTTP Request Node "Get Updates"**  
   - Type: HTTP Request  
   - URL: `https://api.telegram.org/bot{{ $json.bot_token }}/getUpdates`  
   - Query parameters:  
     - `allowed_updates` = `["channel_post"]`  
     - `timeout` = `3` (for long polling)  
     - `offset` = `={{ $json.last_offset }}`  
     - `limit` = `15`  
   - Timeout: 30 seconds  
   - Connect Config node output to this node.

4. **Create Code Node "Process Offset"**  
   - Type: Code (JavaScript)  
   - Use the provided JS code to:  
     - Maintain processed update IDs in workflow global static data.  
     - Filter messages within 15 seconds window.  
     - Update offset tracking.  
     - Return only new messages or an offset placeholder.  
   - Connect Get Updates output to this node.

5. **Create HTTP Request Node "Clear Update List"**  
   - Type: HTTP Request  
   - URL: `https://api.telegram.org/bot{{ $('Config').item.json.bot_token }}/getUpdates?offset={{ $json._current_offset }}`  
   - Same query parameters as Get Updates.  
   - Connect Process Offset node output to this node (parallel to Split node).

6. **Create Split Out Node "Split"**  
   - Type: Split Out  
   - Field to split: `channel_post.text`  
   - Connect Process Offset output to Split node.

7. **Create If Node "Filter"**  
   - Type: If  
   - Condition: Check if `$('Process Offset').item.json.channel_post.chat.id.toString()` equals configured `channel_id` from Config node.  
   - Connect Split output to Filter node.

8. **Create Switch Node "Switch"**  
   - Type: Switch  
   - Conditions:  
     - Output key `am`: message text starts with `am# `  
     - Output key `ami`: message text starts with `ami# `  
   - Connect Filter true output to Switch node.

9. **Text Generation Pipeline**

   a. **Execute Command Node "Execute - Text"**  
      - Install packages: `apk add util-linux-misc` and `apk add curl` if missing.  
      - Download TGPT binary from https://github.com/aandrew-me/tgpt/releases/download/v2.11.0/tgpt-linux-amd64  
      - Rename to `tgpt`, make executable.  
      - Remove `am#` prefix from message text to form prompt.  
      - Run TGPT CLI: `./tgpt --model "gtp-4" --temperature "0.3" -q "$prompt"` output to `/tmp/response.txt`.  
      - Filter out lines starting with `@web_search`.  
      - Continue on fail enabled.  
      - Connect Switch output `am` to this node.

   b. **Code Node "Clean"**  
      - Extract `stdout` from Execute - Text output.  
      - Return cleaned human-readable text or error message.  
      - Connect Execute - Text output to this node.

   c. **Telegram Node "Send Telegram Text Response"**  
      - Operation: Send message  
      - Text: Use cleaned human-readable text from Clean node.  
      - Chat ID: Use configured `channel_id`.  
      - Additional fields: parse mode `HTML`, append attribution false.  
      - Use Telegram API credentials configured with bot token.  
      - Connect Clean output to this node.

10. **Image Generation Pipeline**

    a. **Execute Command Node "Execute - Image"**  
       - Install `util-linux-misc` if missing.  
       - Download TGPT binary as above.  
       - Remove `ami#` prefix from message text to form prompt.  
       - Run TGPT CLI with image generation:  
         `./tgpt -image --height=1080 --width=1920 --out=/tmp/genimg.jpg --model "gtp-4" --temperature "0.7" "$prompt"`  
       - Continue on fail enabled.  
       - Connect Switch output `ami` to this node.

    b. **Read/Write File Node "Read Generated Image"**  
       - File path: `/tmp/genimg.jpg`  
       - Data property name: `genimg` (binary)  
       - Connect Execute - Image output to this node.

    c. **Telegram Node "Send Telegram Image Response"**  
       - Operation: Send Photo  
       - Chat ID: from Config node.  
       - Binary property name: `genimg`  
       - Use same Telegram API credentials as text response node.  
       - Connect Read Generated Image output to this node.

11. **Connect nodes accordingly as per the above connections.**

12. **Configure Telegram Credentials**  
    - Create Telegram API credentials in n8n with bot token from @BotFather.  
    - Apply same credentials to both Telegram send nodes.  
    - Ensure bot is admin in target Telegram channel.

13. **Save and activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                      | Context or Link                                                                                          |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Automated bot monitors Telegram channel, generating AI responses via TGPT with GPT-4, supporting text and image generation commands with prefixes `am#` and `ami#`. | Workflow description and overview.                                                                      |
| Configuration must include valid Telegram bot token and channel ID; bot must be admin. Offset tracking is automatic, ensuring no duplicate message processing.     | Sticky Note1 (Configuration instructions).                                                             |
| Message polling uses Telegram API's `getUpdates` with a 15-second time window filter and duplicate prevention mechanism to avoid reprocessing old messages.      | Sticky Note2 (Polling & processing explanation).                                                        |
| Messages are filtered to ensure origin from configured channel; routing differentiates text vs. image commands based on message prefixes.                       | Sticky Note3 (Filtering and routing explanation).                                                       |
| Text generation pipeline installs TGPT, removes prefix from prompt, and uses GPT-4 with temperature 0.3 to create focused AI text responses.                    | Sticky Note4 (Text generation pipeline details).                                                        |
| Image generation pipeline installs TGPT and generates 1920x1080 images with temperature 0.7 for creative outputs, reading and sending generated image files.    | Sticky Note5 (Image generation pipeline details).                                                       |
| Telegram send nodes require valid credentials created with bot token; messages are sent with HTML parse mode enabled.                                           | Sticky Note6 (Telegram credentials instructions).                                                       |
| Usage examples: `am# Explain quantum physics` for text, `ami# Futuristic city at sunset` for images.                                                           | Sticky Note7 (Usage commands).                                                                           |
| TGPT CLI tool release used: v2.11.0 from https://github.com/aandrew-me/tgpt/releases/tag/v2.11.0                                                                | External dependency. Ensure network access to GitHub for binary download.                                |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This workflow strictly complies with content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.
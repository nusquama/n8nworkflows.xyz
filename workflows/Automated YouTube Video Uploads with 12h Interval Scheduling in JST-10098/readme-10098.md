Automated YouTube Video Uploads with 12h Interval Scheduling in JST

https://n8nworkflows.xyz/workflows/automated-youtube-video-uploads-with-12h-interval-scheduling-in-jst-10098


# Automated YouTube Video Uploads with 12h Interval Scheduling in JST

### 1. Workflow Overview

This workflow automates the process of uploading a series of local video files to YouTube with scheduled publishing times spaced 12 hours apart in Japan Standard Time (JST). It is designed for content creators who want to batch process and schedule their video uploads systematically, ensuring consistent release intervals without manual intervention.

The workflow consists of the following logical blocks:

- **1.1 Input Reception:** Manual trigger to start the workflow execution.
- **1.2 File Retrieval and Preparation:** List all local video files from a specified directory and prepare metadata for each.
- **1.3 Sorting and Scheduling:** Sort videos by extracted day numbers in filenames and calculate scheduled publishing times at 12-hour intervals in JST.
- **1.4 Batch Processing:** Split the video items into single-item batches to handle uploads one at a time.
- **1.5 Video Upload and Playlist Management:** Read each video file, upload it to YouTube as a scheduled private video, and add the uploaded video to a predefined YouTube playlist.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
Starts the workflow manually on demand.

**Nodes Involved:**  
- Manual Trigger

**Node Details:**

- **Manual Trigger**  
  - Type: Trigger node; initiates workflow manually.  
  - Configuration: Default, no parameters.  
  - Inputs: None  
  - Outputs: Connected to "List Video Files" node.  
  - Edge cases: None; manual start depends on user action.

---

#### 1.2 File Retrieval and Preparation

**Overview:**  
Retrieves all `.mp4` video files from a specified local directory and prepares workflow items including file paths and metadata for further processing.

**Nodes Involved:**  
- List Video Files  
- Sort and Generate Items

**Node Details:**

- **List Video Files**  
  - Type: Execute Command  
  - Role: Runs a shell command to find all `.mp4` files under `/opt/downloads/单词卡/A1-A2`.  
  - Configuration: Command uses `find` with `-print0` to safely handle filenames with spaces or special characters.  
  - Inputs: From Manual Trigger  
  - Outputs: Raw stdout with null-separated file paths to "Sort and Generate Items".  
  - Edge cases:  
    - Empty directory returns empty output (no files).  
    - Permission errors or command failure would cause node failure.

- **Sort and Generate Items**  
  - Type: Code (JavaScript)  
  - Role: Parses the null-separated file paths, extracts "day" numbers from filenames using regex, sorts files by day number and filename, and outputs structured items with `path`, `filename`, `day`, and `order` fields.  
  - Configuration:  
    - Uses regex `/day[-_ ]*(\d+)/i` to extract day numbers from filenames.  
    - Sorts ascending by day, then filename.  
  - Inputs: Raw file list from "List Video Files"  
  - Outputs: Sorted, structured video items to "Calculate Publish Schedule (+12h Interval)"  
  - Edge cases:  
    - Files without day number are sorted last (day = Infinity).  
    - Malformed filenames will be sorted but may disrupt intended order.

---

#### 1.3 Sorting and Scheduling

**Overview:**  
Calculates scheduled publishing timestamps for each video spaced 12 hours apart starting from the next whole hour plus a 30-minute buffer in JST timezone.

**Nodes Involved:**  
- Calculate Publish Schedule (+12h Interval)

**Node Details:**

- **Calculate Publish Schedule (+12h Interval)**  
  - Type: Code (JavaScript)  
  - Role: For each video item, calculates `publishAtUtc` ISO timestamp for YouTube scheduled publishing.  
  - Configuration:  
    - Interval: 12 hours (SPAN_HOURS = 12)  
    - Timezone offset for JST: +9 hours  
    - Buffer: 30 minutes after the next whole JST hour to allow for platform processing.  
  - Logic:  
    - Gets current time in UTC, converts to JST, aligns to next hour plus buffer.  
    - Adds offset per video based on order index (0, 1, 2, ...).  
    - Converts back to UTC ISO string for YouTube.  
  - Inputs: Sorted video items from "Sort and Generate Items"  
  - Outputs: Items enriched with `publishAtUtc`, `publishAtLocal` (JST ISO string), `filename`, and `order` to "Split in Batches (1 per video)"  
  - Edge cases:  
    - Invalid or missing order/index causes error and failure.  
    - System clock inaccuracies may affect schedule slightly.

---

#### 1.4 Batch Processing

**Overview:**  
Processes video items one by one to avoid memory overload and simplify error handling during upload.

**Nodes Involved:**  
- Split in Batches (1 per video)

**Node Details:**

- **Split in Batches (1 per video)**  
  - Type: SplitInBatches  
  - Role: Splits input array into batches of size 1 for sequential processing.  
  - Configuration: Default batch size = 1  
  - Inputs: Scheduled video items from "Calculate Publish Schedule (+12h Interval)"  
  - Outputs: Each batch passes to "Read Video File" individually; also outputs all items for looping back to "Add to Playlist".  
  - Edge cases:  
    - Large number of videos increases workflow duration proportionally.  
    - Failure in one batch doesn't affect others directly.

---

#### 1.5 Video Upload and Playlist Management

**Overview:**  
Reads each video file from disk, uploads it to YouTube as a scheduled private video with metadata, then adds the uploaded video to a specific YouTube playlist.

**Nodes Involved:**  
- Read Video File  
- Upload to YouTube (Scheduled)  
- Add to Playlist

**Node Details:**

- **Read Video File**  
  - Type: ReadWriteFile (Read mode)  
  - Role: Reads the video file from disk for upload.  
  - Configuration: Reads file from path `/opt/downloads/单词卡/A1-A2/{{ $json.filename }}` dynamically per item.  
  - Inputs: Single video batch from "Split in Batches (1 per video)"  
  - Outputs: Binary file data to "Upload to YouTube (Scheduled)"  
  - Edge cases:  
    - File not found or permission denied causes failure.  
    - Large files may increase memory usage.

- **Upload to YouTube (Scheduled)**  
  - Type: YouTube node  
  - Role: Uploads video as private, scheduled to be published at calculated `publishAtUtc`.  
  - Configuration:  
    - Title fixed as: `A1–A2单词打卡计划 #轻松学英语 #英语学习 #英语单词 #英语打卡 #零基础英语`  
    - Description fixed with hashtags and content description in Chinese and English.  
    - Tags: `轻松学英语,英语学习,英语单词,英语打卡,零基础英语`  
    - CategoryId: 27 (Education)  
    - RegionCode: CN  
    - PrivacyStatus: private  
    - PublishAt: dynamic, from batch item `publishAtUtc`  
  - Credentials: YouTube OAuth2 (authenticated account)  
  - Inputs: Video file binary and metadata from "Read Video File"  
  - Outputs: Returns uploaded video's YouTube `uploadId` (video ID) to "Add to Playlist"  
  - Edge cases:  
    - API quota limits or authentication failures.  
    - Network timeouts or upload interruptions.  
    - Incorrect `publishAt` ISO format causes API errors.

- **Add to Playlist**  
  - Type: YouTube node  
  - Role: Adds uploaded video to a predefined YouTube playlist.  
  - Configuration:  
    - Playlist ID fixed: `PLJ3aD-smyb90ZmBJ9jcuW7AcnGeHF_jfF`  
    - VideoId dynamically assigned from previous node’s upload ID.  
  - Credentials: Same YouTube OAuth2 account.  
  - Inputs: Video upload response from "Upload to YouTube (Scheduled)"  
  - Outputs: Passes output back to "Split in Batches (1 per video)" for next batch processing.  
  - Edge cases:  
    - Playlist ID invalid or access denied.  
    - API failures or quota limits.

---

### 3. Summary Table

| Node Name                      | Node Type               | Functional Role                          | Input Node(s)                 | Output Node(s)                  | Sticky Note                                                                                 |
|--------------------------------|-------------------------|----------------------------------------|------------------------------|-------------------------------|----------------------------------------------------------------------------------------------|
| Manual Trigger                 | Manual Trigger          | Workflow start trigger                  | None                         | List Video Files               |                                                                                              |
| List Video Files              | Execute Command         | Lists all local video files             | Manual Trigger               | Sort and Generate Items        | ## List & Prepare Files<br>Retrieves all video files from the local folder and prepares them. |
| Sort and Generate Items       | Code (JavaScript)       | Parses and sorts video file list        | List Video Files             | Calculate Publish Schedule (+12h Interval) | ## Sort & Schedule Uploads<br>Sorts videos and calculates upload schedule.                    |
| Calculate Publish Schedule (+12h Interval) | Code (JavaScript)       | Calculates scheduled publish times      | Sort and Generate Items      | Split in Batches (1 per video)  |                                                                                              |
| Split in Batches (1 per video) | SplitInBatches          | Processes videos one at a time           | Calculate Publish Schedule (+12h Interval) | Read Video File (batch output), Add to Playlist (all items output) | ## Process in Batches<br>Ensures one-by-one processing for memory and error control.           |
| Read Video File               | ReadWriteFile           | Reads video file from disk               | Split in Batches (1 per video) | Upload to YouTube (Scheduled)  |                                                                                              |
| Upload to YouTube (Scheduled) | YouTube                 | Uploads video as scheduled private video | Read Video File              | Add to Playlist                | ## Upload to YouTube<br>Uploads video scheduled for future publishing.                        |
| Add to Playlist               | YouTube                 | Adds uploaded video to a YouTube playlist | Upload to YouTube (Scheduled) | Split in Batches (1 per video) | ## Add to Playlist<br>Adds video to specified playlist for channel organization.             |
| Sticky Note                   | Sticky Note             | Overview note                           | None                         | None                          | # Overview<br>This workflow automates video publishing to YouTube.                           |
| Sticky Note1                  | Sticky Note             | List & prepare files note               | None                         | None                          | ## List & Prepare Files<br>Retrieves all video files and prepares metadata.                   |
| Sticky Note2                  | Sticky Note             | Sort & schedule uploads note            | None                         | None                          | ## Sort & Schedule Uploads<br>Sorts videos and calculates spaced publish times.              |
| Sticky Note3                  | Sticky Note             | Process in batches note                  | None                         | None                          | ## Process in Batches<br>Handles videos sequentially to avoid overload.                       |
| Sticky Note4                  | Sticky Note             | Upload to YouTube note                   | None                         | None                          | ## Upload to YouTube<br>Uploads scheduled videos as private with future publish time.        |
| Sticky Note5                  | Sticky Note             | Add to playlist note                     | None                         | None                          | ## Add to Playlist<br>Adds uploaded videos to the chosen YouTube playlist.                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a "Manual Trigger" node:**  
   - No configuration needed. This node will start the workflow manually.

3. **Add an "Execute Command" node named "List Video Files":**  
   - Connect its input to "Manual Trigger".  
   - Set the command to:  
     `find /opt/downloads/单词卡/A1-A2 -type f -name '*.mp4' -print0`  
   - This lists all `.mp4` files under the specified directory, outputting null-separated paths.

4. **Add a "Code" node named "Sort and Generate Items":**  
   - Connect input from "List Video Files".  
   - Use the following JavaScript code:  
     ```js
     const raw = $input.first().json.stdout ?? '';
     const paths = raw.split('\u0000').filter(Boolean);

     function extractDayNum(p) {
       const name = p.split('/').pop();
       const m = name.match(/day[-_ ]*(\d+)/i);
       return m ? parseInt(m[1], 10) : Number.POSITIVE_INFINITY;
     }

     const sorted = paths
       .filter(p => p.toLowerCase().endsWith('.mp4'))
       .map(p => ({
         path: p,
         filename: p.split('/').pop(),
         day: extractDayNum(p),
       }))
       .sort((a, b) => (a.day - b.day) || a.filename.localeCompare(b.filename));

     return sorted.map((it, i) => ({
       json: {
         path: it.path,
         filename: it.filename,
         day: it.day,
         order: i
       }
     }));
     ```
   - This parses and sorts the files.

5. **Add a "Code" node named "Calculate Publish Schedule (+12h Interval)":**  
   - Connect input from "Sort and Generate Items".  
   - Set mode to "runOnceForEachItem".  
   - Use the following JavaScript code:  
     ```js
     const SPAN_HOURS   = 12;
     const TZ_OFFSET_MS = 9 * 3600e3;
     const BUFFER_MIN   = 30;

     const rawOrder = $json.order ?? $json.index ?? 0;
     const order = Number(rawOrder);
     if (!Number.isFinite(order) || order < 0) {
       throw new Error(`Invalid order/index: ${rawOrder}`);
     }

     const nowUtcMs   = Date.now();
     const nowJstMs   = nowUtcMs + TZ_OFFSET_MS;

     const HOUR_MS    = 3600e3;
     const MIN_MS     = 60e3;
     const nextHourJstMs = Math.ceil(nowJstMs / HOUR_MS) * HOUR_MS;

     const baseJstMs  = nextHourJstMs + BUFFER_MIN * MIN_MS;

     const publishJstMs = baseJstMs + order * SPAN_HOURS * HOUR_MS;

     const publishUtcMs  = publishJstMs - TZ_OFFSET_MS;
     const publishAtUtc  = new Date(publishUtcMs).toISOString();

     const publishAtLocal = new Date(publishJstMs).toISOString().slice(0,19) + '+09:00';

     return {
       publishAtUtc,
       publishAtLocal,
       filename: ($json.path ?? '').split('/').pop(),
       order
     };
     ```
   - This calculates the scheduled publish time per video.

6. **Add a "SplitInBatches" node named "Split in Batches (1 per video)":**  
   - Connect input from "Calculate Publish Schedule (+12h Interval)".  
   - Set batch size to 1 (default).  
   - This ensures videos are processed sequentially.

7. **Add a "Read Binary File" node named "Read Video File":**  
   - Connect input from the second output of "Split in Batches (1 per video)" (the current batch).  
   - Set mode to read file.  
   - Set file path expression to:  
     `/opt/downloads/单词卡/A1-A2/{{ $json.filename }}`  
   - Enable "Retry On Fail" to handle transient errors.

8. **Add a "YouTube" node named "Upload to YouTube (Scheduled)":**  
   - Connect input from "Read Video File".  
   - Set operation to `upload` and resource to `video`.  
   - Configure video metadata:  
     - Title: `A1–A2单词打卡计划 #轻松学英语 #英语学习 #英语单词 #英语打卡 #零基础英语`  
     - Description:  
       `A1–A2 英语单词学习系列 | 零基础英语 | 每天10分钟掌握核心词汇 帮助初学者系统学习基础英语词汇，从A1到A2逐步提升。 轻松学英语、英语打卡、自学复习必备。 #英语学习 #英语单词 #零基础英语 #EnglishVocabulary #A1A2English`  
     - Tags: `轻松学英语,英语学习,英语单词,英语打卡,零基础英语`  
     - Privacy status: `private`  
     - PublishAt: Expression: `={{ $('Split in Batches (1 per video)').item.json.publishAtUtc }}`  
     - Category ID: `27` (Education)  
     - Region Code: `CN`  
   - Set YouTube OAuth2 credentials (create or select OAuth2 credentials for YouTube API).  
   - Enable "Retry On Fail" with a 3-second wait between retries.

9. **Add a "YouTube" node named "Add to Playlist":**  
   - Connect input from "Upload to YouTube (Scheduled)".  
   - Set resource to `playlistItem` and operation to `create`.  
   - Playlist ID: `PLJ3aD-smyb90ZmBJ9jcuW7AcnGeHF_jfF`  
   - Video ID: Expression: `={{ $json.uploadId }}` (from upload response)  
   - Use the same YouTube OAuth2 credentials as above.  
   - Enable "Retry On Fail" with a 3-second wait.

10. **Connect the output of "Add to Playlist" back to the first output of "Split in Batches (1 per video)" to continue processing the rest of the batch items.**

11. **Optionally, add Sticky Notes and comments at each logical block for clarity.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                         | Context or Link                                                                                      |
|--------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow automates video publishing to YouTube with a 12-hour interval scheduling in JST timezone.                             | Overview sticky note in the workflow UI.                                                           |
| The YouTube category ID `27` corresponds to Education content.                                                                        | YouTube API documentation: https://developers.google.com/youtube/v3/docs/videoCategories/list       |
| The workflow uses a buffer of 30 minutes after the next full JST hour to schedule uploads, allowing platform processing time.       | Scheduling logic in "Calculate Publish Schedule (+12h Interval)" node.                              |
| Videos are uploaded as private and automatically become public at the scheduled `publishAt` time.                                   | YouTube upload settings in "Upload to YouTube (Scheduled)".                                        |
| Playlist ID `PLJ3aD-smyb90ZmBJ9jcuW7AcnGeHF_jfF` is hardcoded; replace with your own playlist ID to organize uploaded content.     | "Add to Playlist" node configuration.                                                              |
| The workflow handles large files by processing videos one at a time (batch size 1) to avoid memory overload and ease error recovery.| SplitInBatches node configuration and sticky note.                                                 |
| Ensure n8n instance has access to the local video directory and sufficient permissions to read files.                               | File system access considerations.                                                                 |
| Use YouTube OAuth2 credentials with appropriate scopes for video upload and playlist management.                                    | Credential configuration for YouTube OAuth2 API.                                                  |

---

**Disclaimer:**  
The provided text is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and publicly accessible.
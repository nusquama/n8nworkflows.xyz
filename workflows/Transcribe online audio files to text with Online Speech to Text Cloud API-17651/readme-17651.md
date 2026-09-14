Transcribe online audio files to text with Online Speech to Text Cloud API

https://n8nworkflows.xyz/workflows/transcribe-online-audio-files-to-text-with-online-speech-to-text-cloud-api-17651


# Transcribe online audio files to text with Online Speech to Text Cloud API

### 1. Workflow Overview

This workflow is designed to manually initiate, monitor, and retrieve asynchronous speech-to-text transcriptions for online audio files using the "Online Speech to Text Cloud API" community nodes. It targets use cases requiring the conversion of long-form audio files into written text documents where processing time prevents synchronous responses.

The logical execution is grouped into four sequential blocks:
- **1.1 Input Reception & Parameter Setup:** Initializes the workflow execution manually and declares required variables such as the target audio URL, language code, and expected output format.
- **1.2 Audio Retrieval & Job Submission:** Downloads the raw audio binary file from the specified URL and submits it to the Speech-to-Text Cloud API to create a background transcription job.
- **1.3 Polling & Status Verification:** Enters a loop utilizing a wait node and a status check request to monitor job progress until completion.
- **1.4 Transcript Retrieval:** Fetches and outputs the final completed text transcript once the job status indicates success.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Parameter Setup
- **Overview:** Starts the workflow on-demand and defines configuration parameters necessary for the audio fetching and transcription steps.
- **Nodes Involved:** `When Started Manually`, `Set Audio Parameters`
- **Node Details:**
  - **When Started Manually**
    - *Type and Role:* `n8n-nodes-base.manualTrigger` (Trigger node). Starts the execution pipeline upon user click.
    - *Configuration:* Default settings.
    - *Input/Output:* No inputs; outputs to `Set Audio Parameters`.
    - *Edge Cases:* None.
  - **Set Audio Parameters**
    - *Type and Role:* `n8n-nodes-base.set` (Data transformation node). Establishes static operational variables.
    - *Configuration:* Assigns three string variables: `audio_url`, `language`, and `output_format`.
    - *Key Expressions:* 
      - `audio_url`: `https://www.raspberry-fertig.de/cloud/public.php/dav/files/H2dJKABksbMykrT/`
      - `language`: `yyy` (Placeholder language code)
      - `output_format`: `txt`
    - *Input/Output:* Input from `When Started Manually`; output to `Fetch Audio from URL`.
    - *Edge Cases:* Invalid URLs or unreachable file endpoints will cause subsequent HTTP download failures.

#### 2.2 Audio Retrieval & Job Submission
- **Overview:** Downloads the binary audio data from the designated URL and sends it to the cloud transcription engine to generate a job identifier.
- **Nodes Involved:** `Fetch Audio from URL`, `Transcribe Audio File`
- **Node Details:**
  - **Fetch Audio from URL**
    - *Type and Role:* `n8n-nodes-base.httpRequest` (Network request node). Downloads the external media file.
    - *Configuration:* Makes a GET request to download binary content.
    - *Key Expressions:* URL set via expression `={{ $json.audio_url }}`.
    - *Input/Output:* Input from `Set Audio Parameters`; output to `Transcribe Audio File`.
    - *Edge Cases:* Timeouts on large audio files, HTTP 401/403 errors if the source URL requires authentication, or DNS resolution failures.
  - **Transcribe Audio File**
    - *Type and Role:* `n8n-nodes-speech-to-text-cloud.speechToTextCloud` (Community action node). Submits the audio file for asynchronous processing.
    - *Configuration:* Configured with the specified language parameter and utilizes API credentials.
    - *Key Expressions:* Language: `={{ $json.language }}`.
    - *Credentials:* `Speech To Text Cloud account` (`speechToTextCloudApi`).
    - *Input/Output:* Input from `Fetch Audio from URL`; output to `Wait for Transcription`.
    - *Version-specific requirements:* Requires the community node package `n8n-nodes-speech-to-text-cloud` installed on the n8n instance.
    - *Edge Cases:* API authentication errors, unsupported audio formats, or payload size limits exceeded.

#### 2.3 Polling & Status Verification
- **Overview:** Introduces a pause between API calls, checks the active job status, and loops back if processing is still ongoing.
- **Nodes Involved:** `Wait for Transcription`, `Fetch Transcription Status`, `If Transcription Complete`
- **Node Details:**
  - **Wait for Transcription**
    - *Type and Role:* `n8n-nodes-base.wait` (Flow control node). Pauses workflow execution for a configured duration before checking status.
    - *Configuration:* Uses default wait settings or webhook resumption (configured via resume criteria).
    - *Input/Output:* Input from `Transcribe Audio File` (and loop-back from `If Transcription Complete`); output to `Fetch Transcription Status`.
    - *Edge Cases:* Excessive API calls if wait duration is too short, leading to rate limiting.
  - **Fetch Transcription Status**
    - *Type and Role:* `n8n-nodes-speech-to-text-cloud.speechToTextCloud` (Community action node). Queries the API for current job metrics.
    - *Configuration:* Operation set to `get` using the job ID from the initial submission.
    - *Key Expressions:* `jobId`: `={{ $('Transcribe Audio File').item.json.job_id }}`.
    - *Credentials:* `Speech To Text Cloud account` (`speechToTextCloudApi`).
    - *Input/Output:* Input from `Wait for Transcription`; output to `If Transcription Complete`.
    - *Edge Cases:* Expired job IDs or network communication drops.
  - **If Transcription Complete**
    - *Type and Role:* `n8n-nodes-base.if` (Conditional routing node). Evaluates whether the transcription job has successfully finished.
    - *Configuration:* Uses strict type validation and binary condition logic. (Note: Left and right condition values require explicit configuration mapping to API response fields).
    - *Input/Output:* Input from `Fetch Transcription Status`; output True branch to `Retrieve Final Transcript`, False branch loops back to `Wait for Transcription`.
    - *Edge Cases:* Infinite loops if the API status property names mismatch condition rules or if jobs fail permanently without reporting a terminal state.

#### 2.4 Transcript Retrieval
- **Overview:** Requests and outputs the final generated text document once confirmation is received.
- **Nodes Involved:** `Retrieve Final Transcript`
- **Node Details:**
  - **Retrieve Final Transcript**
    - *Type and Role:* `n8n-nodes-speech-to-text-cloud.speechToTextCloud` (Community action node). Downloads the finalized transcript payload.
    - *Configuration:* Operation set to `get` targeting the original job reference.
    - *Key Expressions:* `jobId`: `={{ $('Transcribe Audio File').item.json.job_id }}`.
    - *Credentials:* `Speech To Text Cloud account` (`speechToTextCloudApi`).
    - *Input/Output:* Input from `If Transcription Complete` (True branch); no downstream outputs.
    - *Edge Cases:* Missing output formatting parameters or corrupted payload returns.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | `n8n-nodes-base.stickyNote` | Documentation wrapper | None | None | ## Transcribe Audio Files Using Online Speech to Text Cloud API<br><br>### How it works<br><br>This workflow manually starts an audio transcription job using an online Speech to Text Cloud API. It sets the audio URL and transcription options, downloads the audio file, submits it for transcription, then polls the job status until processing is finished. Once complete, it retrieves the final transcript.<br><br>### Setup steps<br><br>- Configure credentials for the Speech to Text Cloud API nodes.<br>- Set the `audio_url`, `language`, and `output_format` values in the Set Audio Parameters node.<br>- Ensure the audio URL is reachable by the workflow, or configure authentication in the Fetch Audio File HTTP request if the file is private.<br>- Adjust the wait duration and status condition to match the API's expected processing time and response format.<br><br>### Customization<br><br>Change the language, output format, polling interval, or completion condition to fit the speech-to-text provider and desired transcript format. |
| Sticky Note1 | `n8n-nodes-base.stickyNote` | Documentation wrapper | None | None | ## Manual input setup<br><br>Starts the workflow manually and defines the audio URL plus transcription options that the rest of the workflow will use. |
| Sticky Note2 | `n8n-nodes-base.stickyNote` | Documentation wrapper | None | None | ## Fetch and submit audio<br><br>Downloads the configured audio file and sends it to the Speech to Text Cloud API to start a transcription job. |
| Sticky Note3 | `n8n-nodes-base.stickyNote` | Documentation wrapper | None | None | ## Poll transcription status<br><br>Waits between checks, retrieves the current transcription job status, and loops back until the job is marked as finished. |
| Sticky Note4 | `n8n-nodes-base.stickyNote` | Documentation wrapper | None | None | ## Retrieve final transcript<br><br>Fetches the completed transcription result from the Speech to Text Cloud API after the status check succeeds. |
| When Started Manually | `n8n-nodes-base.manualTrigger` | Manual Execution Trigger | None | Set Audio Parameters | |
| Set Audio Parameters | `n8n-nodes-base.set` | Define variables | When Started Manually | Fetch Audio from URL | |
| Fetch Audio from URL | `n8n-nodes-base.httpRequest` | Download Audio Binary | Set Audio Parameters | Transcribe Audio File | |
| Transcribe Audio File | `n8n-nodes-speech-to-text-cloud.speechToTextCloud` | Submit Transcription Job | Fetch Audio from URL | Wait for Transcription | |
| Wait for Transcription | `n8n-nodes-base.wait` | Pause Execution | Transcribe Audio File | Fetch Transcription Status | |
| Fetch Transcription Status | `n8n-nodes-speech-to-text-cloud.speechToTextCloud` | Query Job Status | Wait for Transcription | If Transcription Complete | |
| If Transcription Complete | `n8n-nodes-base.if` | Conditional Loop Control | Fetch Transcription Status | Retrieve Final Transcript, Wait for Transcription | |
| Retrieve Final Transcript | `n8n-nodes-speech-to-text-cloud.speechToTextCloud` | Fetch Final Output | If Transcription Complete | None | |

---

### 4. Reproducing the Workflow from Scratch

1. **Install Prerequisites:** Ensure your n8n environment supports community nodes and install the package `n8n-nodes-speech-to-text-cloud`.
2. **Create Credentials:** Set up a new credential of type `speechToTextCloudApi` named "Speech To Text Cloud account" using your valid API key.
3. **Add Trigger:** Create a **When Started Manually** (`n8n-nodes-base.manualTrigger`) node.
4. **Configure Parameters:** Create a **Set Audio Parameters** (`n8n-nodes-base.set`) node. Connect `When Started Manually` to it. Add three assignments (String type):
   - `audio_url`: `https://www.raspberry-fertig.de/cloud/public.php/dav/files/H2dJKABksbMykrT/`
   - `language`: `yyy`
   - `output_format`: `txt`
5. **Add HTTP Request:** Create a **Fetch Audio from URL** (`n8n-nodes-base.httpRequest`) node. Connect `Set Audio Parameters` to it. Set the URL parameter to `={{ $json.audio_url }}`.
6. **Add Transcription Submission Node:** Create a **Transcribe Audio File** (`n8n-nodes-speech-to-text-cloud.speechToTextCloud`) node. Connect `Fetch Audio from URL` to it. Link your `Speech To Text Cloud account` credential. Set `language` to `={{ $json.language }}`.
7. **Add Wait Node:** Create a **Wait for Transcription** (`n8n-nodes-base.wait`) node. Connect `Transcribe Audio File` to it. Keep default pause configuration parameters.
8. **Add Status Check Node:** Create a **Fetch Transcription Status** (`n8n-nodes-speech-to-text-cloud.speechToTextCloud`) node. Connect `Wait for Transcription` to it. Link your `Speech To Text Cloud account` credential, select operation `get`, and set `jobId` to `={{ $('Transcribe Audio File').item.json.job_id }}`.
9. **Add Conditional Evaluation:** Create an **If Transcription Complete** (`n8n-nodes-base.if`) node. Connect `Fetch Transcription Status` to it. Configure evaluation rules matching your API's completion response criteria.
10. **Configure Loop-Back:** Connect the *False* (bottom) output of **If Transcription Complete** back into the input of **Wait for Transcription**.
11. **Add Final Retrieval Node:** Create a **Retrieve Final Transcript** (`n8n-nodes-speech-to-text-cloud.speechToTextCloud`) node. Connect the *True* (top) output of **If Transcription Complete** to it. Link your `Speech To Text Cloud account` credential, select operation `get`, and set `jobId` to `={{ $('Transcribe Audio File').item.json.job_id }}`.
12. **Add Documentation:** Optionally, create and position Sticky Notes around functional clusters to mirror the structural design layout.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Disclaimer: Processed via automated n8n tooling in compliance with content policies using public/legal data. | Workflow metadata and generation context |
Create batch faceless video voice-overs with Google Gemini and Telegram

https://n8nworkflows.xyz/workflows/create-batch-faceless-video-voice-overs-with-google-gemini-and-telegram-20491


# Create batch faceless video voice-overs with Google Gemini and Telegram

### 1. Workflow Overview

This workflow automates the end-to-end production of faceless video voice-overs. It retrieves queued topics from an n8n Data Table, uses Google Gemini to generate and validate structured video scripts, synthesizes narration using Gemini Text-to-Speech (TTS), performs optional transcription quality assurance (listen-back), and dispatches the resulting audio and script files to Telegram for human approval or rejection.

The workflow logic is divided into the following functional blocks:

- **1.1 Initialization and Queue Retrieval:** Schedules execution, initializes channel settings, ensures the Data Table exists, and fetches pending topics.
- **1.2 Script Generation and Validation:** Formats niche-specific system prompts, invokes Google Gemini models with fallback handling, and evaluates script metrics (length, hook strength, scene count, and facts).
- **1.3 Automated Script Repair:** Identifies structural script flaws, invokes a secondary repair prompt if needed, and selects the optimal script version.
- **1.4 Voice Synthesis and Scene Timing:** Generates audio files via Gemini TTS with fallback support, analyzes WAV file markers to compute exact pause timings, creates SRT captions, and packages metadata.
- **1.5 Transcription Quality Assurance:** Optionally transcribes the generated audio back via Gemini and evaluates word-level match accuracy against the original script.
- **1.6 Database Persistence and Review Dispatch:** Updates the Data Table with analysis results and spawns sub-workflow execution runs for individual item reviews.
- **1.7 Telegram Review and Approval Loop:** Renders structured review cards, transmits audio and text assets to Telegram, manages user approval or redo requests, and updates queue statuses accordingly.
- **1.8 Topic Ingestion Form:** Provides an external form trigger to populate the queue with new topics dynamically.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Initialization and Queue Retrieval
**Overview:**  
Sets up the runtime environment by establishing execution intervals, applying global channel configurations, ensuring the required database table exists, and fetching batch items marked for production.

- **Nodes Involved:**
  - `Run Every Morning`
  - `Set Channel Settings`
  - `Create Queue Table If Missing`
  - `Get Next Queued Topics`
  - `Check Queue Has Topics`
  - `Notify Empty Queue`

- **Node Details:**
  - **Run Every Morning**
    - *Type and Technical Role:* Schedule Trigger node executing daily at 07:00.
    - *Configuration Choices:* Configured to fire once daily at hour 7.
    - *Input/Output:* No inputs; outputs to `Set Channel Settings`.
    - *Failure Types:* None; missed runs execute upon next system wake if catch-up is enabled.
  - **Set Channel Settings**
    - *Type and Technical Role:* Set (Edit Fields) node defining global operational parameters.
    - *Configuration Choices:* Assigns channel name, angle, Telegram chat ID, batch size (`3`), default language (`en`), TTS models, and approval timeouts.
    - *Input/Output:* Input from `Run Every Morning`; outputs to `Create Queue Table If Missing` and `Get Next Queued Topics`.
    - *Failure Types:* Expression evaluation failures if referenced settings variables are malformed.
  - **Create Queue Table If Missing**
    - *Type and Technical Role:* n8n Data Table management node ensuring data schema availability.
    - *Configuration Choices:* Creates table `vs_queue` with predefined columns (`topic`, `niche`, `format`, `status`, `package_json`, etc.) if absent.
    - *Input/Output:* Input from `Set Channel Settings`; outputs downstream.
    - *Failure Types:* Database permission errors.
  - **Get Next Queued Topics**
    - *Type and Technical Role:* n8n Data Table query node.
    - *Configuration Choices:* Filters rows where `status` equals `queued`, sorted ascending by `publish_date`, limited by the `batch_size` parameter.
    - *Input/Output:* Input from `Set Channel Settings`; outputs to `Check Queue Has Topics`.
    - *Failure Types:* Query timeouts or table access locks.
  - **Check Queue Has Topics**
    - *Type and Technical Role:* If branching node.
    - *Configuration Choices:* Evaluates whether the retrieved topic string is non-empty.
    - *Input/Output:* Input from `Get Next Queued Topics`. True branch goes to `Build Voice Direction Prompt`; False branch goes to `Notify Empty Queue`.
  - **Notify Empty Queue**
    - *Type and Technical Role:* Telegram notification node.
    - *Configuration Choices:* Sends an empty queue alert using the configured `telegram_chat_id`. Error handling continues regular output.
    - *Input/Output:* Input from `Check Queue Has Topics` (false branch).
    - *Failure Types:* Telegram API rate limits or invalid chat IDs.

---

#### Block 1.2: Script Generation and Validation
**Overview:**  
Constructs customized AI direction prompts based on selected niches and formats, queries primary and backup Google Gemini models, and runs an internal JavaScript validation routine on the returned output.

- **Nodes Involved:**
  - `Build Voice Direction Prompt`
  - `Write Script with Gemini`
  - `Write Script with Backup Gemini`
  - `Check Script Quality`
  - `Check Script Exists`
  - `List Topics Without Script`
  - `Notify Missing Scripts`

- **Node Details:**
  - **Build Voice Direction Prompt**
    - *Type and Technical Role:* Code node (JavaScript) compiling formatting libraries, constraints, and content variables.
    - *Configuration Choices:* Injects niche rules, timing constraints, word counts, and structural beats into a structured prompt object.
    - *Input/Output:* Input from `Check Queue Has Topics`; outputs to `Write Script with Gemini`.
  - **Write Script with Gemini**
    - *Type and Technical Role:* LangChain Google Gemini Chat Model node.
    - *Configuration Choices:* Uses model `models/gemini-flash-lite-latest` with temperature `0.9` and structured JSON output. Configured with error fallback connections (`onError: continueErrorOutput`) and 3 max retries.
    - *Input/Output:* Input from `Build Voice Direction Prompt`; outputs to `Check Script Quality` (success) and `Write Script with Backup Gemini` (error branch).
    - *Failure Types:* API timeouts, model quota exhaustion, or unparseable JSON schemas.
  - **Write Script with Backup Gemini**
    - *Type and Technical Role:* LangChain Google Gemini Chat Model node.
    - *Configuration Choices:* Uses backup model `models/gemini-3.1-flash-lite` with identical parameters upon primary failure.
    - *Input/Output:* Input from `Write Script with Gemini` error branch; outputs to `Check Script Quality`.
  - **Check Script Quality**
    - *Type and Technical Role:* Code node analyzing generated script text.
    - *Configuration Choices:* Sanitizes text, strips prohibited cliches, validates word counts against target constraints, and structures check arrays.
    - *Input/Output:* Inputs from script writing nodes; outputs to `Check Script Exists`.
  - **Check Script Exists**
    - *Type and Technical Role:* If branching node.
    - *Configuration Choices:* Checks if `has_script` evaluates to true.
    - *Input/Output:* Input from `Check Script Quality`. True branch flows to `Check Fix Needed`; False branch flows to `List Topics Without Script`.
  - **List Topics Without Script**
    - *Type and Technical Role:* Code node aggregating missing script titles.
    - *Input/Output:* Input from `Check Script Exists` (false branch); outputs to `Notify Missing Scripts`.
  - **Notify Missing Scripts**
    - *Type and Technical Role:* Telegram notification node.
    - *Configuration Choices:* Dispatches warning alerts concerning un-scripted queue rows.
    - *Input/Output:* Input from `List Topics Without Script`.

---

#### Block 1.3: Automated Script Repair
**Overview:**  
Evaluates validation problem sets, triggers a targeted correction pass via secondary AI generation if issues exist, and performs a comparative evaluation to keep the optimal script version.

- **Nodes Involved:**
  - `Check Fix Needed`
  - `Fix Script with Gemini`
  - `Pick Best Script Version`

- **Node Details:**
  - **Check Fix Needed**
    - *Type and Technical Role:* If branching node.
    - *Configuration Choices:* Evaluates whether `problems.length` is greater than `0`.
    - *Input/Output:* Input from `Check Script Exists`; True branch goes to `Fix Script with Gemini`; False branch goes to `Pick Best Script Version`.
  - **Fix Script with Gemini**
    - *Type and Technical Role:* LangChain Google Gemini Chat Model node.
    - *Configuration Choices:* Uses `models/gemini-3.1-flash-lite` with a lower temperature (`0.4`) to resolve specific reported validation errors.
    - *Input/Output:* Input from `Check Fix Needed`; outputs to `Pick Best Script Version`.
    - *Failure Types:* API rejection of repair instruction strings.
  - **Pick Best Script Version**
    - *Type and Technical Role:* Code node evaluating script quality scores.
    - *Configuration Choices:* Compares pre-fix and post-fix problem counts, selecting the version with fewer defects, and builds TTS request payloads.
    - *Input/Output:* Inputs from `Check Fix Needed` (false branch) and `Fix Script with Gemini`; outputs to `Generate Voice with Gemini TTS`.

---

#### Block 1.4: Voice Synthesis and Scene Timing
**Overview:**  
Calls the Gemini TTS endpoints to generate WAV audio streams, parses internal audio data structures, calculates pause-based scene timings, and generates SRT subtitle files.

- **Nodes Involved:**
  - `Generate Voice with Gemini TTS`
  - `Generate Voice with Backup TTS`
  - `Build Voice-over Package`

- **Node Details:**
  - **Generate Voice with Gemini TTS**
    - *Type and Technical Role:* HTTP Request node calling Google Gemini interaction endpoints.
    - *Configuration Choices:* POST request to `https://generativelanguage.googleapis.com/v1beta/interactions` using Google Palm API credentials with a 10-minute timeout.
    - *Input/Output:* Input from `Pick Best Script Version`; outputs to `Build Voice-over Package` (success) and `Generate Voice with Backup TTS` (error branch).
    - *Failure Types:* Network timeouts, audio generation failures, or API rate limits.
  - **Generate Voice with Backup TTS**
    - *Type and Technical Role:* HTTP Request node acting as a secondary synthesis fallback.
    - *Configuration Choices:* Sends identical payloads using the `tts_backup_model` configuration variable.
    - *Input/Output:* Input from `Generate Voice with Gemini TTS` error branch; outputs to `Build Voice-over Package`.
  - **Build Voice-over Package**
    - *Type and Technical Role:* Code node parsing binary audio buffers.
    - *Configuration Choices:* Extracts base64 audio blocks, analyzes WAV headers and RMS levels to isolate real pauses, synchronizes scene boundary markers, builds SRT captions, and outputs binary file payloads.
    - *Input/Output:* Inputs from TTS nodes; outputs to `Check Listen-back Enabled`.

---

#### Block 1.5: Transcription Quality Assurance
**Overview:**  
Optionally transcribes generated audio files back into text via Google Gemini and performs a word-level edit-distance comparison against the original script.

- **Nodes Involved:**
  - `Check Listen-back Enabled`
  - `Transcribe Voice with Gemini`
  - `Compare Transcript with Script`

- **Node Details:**
  - **Check Listen-back Enabled**
    - *Type and Technical Role:* If branching node.
    - *Configuration Choices:* Verifies that `has_audio` is true and `listen_back_check` is enabled in global settings.
    - *Input/Output:* Input from `Build Voice-over Package`. True branch goes to `Transcribe Voice with Gemini`; False branch routes directly to `Compare Transcript with Script`.
  - **Transcribe Voice with Gemini**
    - *Type and Technical Role:* LangChain Google Gemini Audio Analysis node.
    - *Configuration Choices:* Uses `models/gemini-3.1-flash-lite` to ingest the binary WAV attachment and return a verbatim text transcription.
    - *Input/Output:* Input from `Check Listen-back Enabled`; outputs to `Compare Transcript with Script`.
    - *Failure Types:* Audio stream parsing errors or model hallucinations during transcription.
  - **Compare Transcript with Script**
    - *Type and Technical Role:* Code node executing a Levenshtein-based distance algorithm.
    - *Configuration Choices:* Computes accuracy percentage, identifies missing or extra words, structures final package JSON documents, and configures sub-workflow execution parameters.
    - *Input/Output:* Inputs from `Check Listen-back Enabled` (bypass) and `Transcribe Voice with Gemini`; outputs to `Save Results to Queue` and `Send Each Voice-over to Review`.

---

#### Block 1.6: Database Persistence and Review Dispatch
**Overview:**  
Persists analysis metrics and package files back to the `vs_queue` Data Table and spawns isolated sub-workflow executions for each individual voice-over package review.

- **Nodes Involved:**
  - `Save Results to Queue`
  - `Send Each Voice-over to Review`

- **Node Details:**
  - **Save Results to Queue**
    - *Type and Technical Role:* n8n Data Table row update node.
    - *Configuration Choices:* Updates table row matching `row_id` with execution metadata, title, status (`review`), duration, listen-back score, and serialized package JSON.
    - *Input/Output:* Input from `Compare Transcript with Script`; terminal branch.
    - *Failure Types:* Database update collisions or row ID mismatches.
  - **Send Each Voice-over to Review**
    - *Type and Technical Role:* Execute Workflow node (Sub-workflow invocation).
    - *Configuration Choices:* Invokes the workflow itself (`$workflow.id`) in `each` mode with `waitForSubWorkflow: false` to process item reviews independently.
    - *Input/Output:* Input from `Compare Transcript with Script`; initiates parallel review runs.

---

#### Block 1.7: Telegram Review and Approval Loop
**Overview:**  
Acts as the execution handler for individual voice-over reviews. Formats rich Telegram review cards, attaches WAV and TXT assets, dispatches interactive approval buttons, and updates queue rows based on user decisions.

- **Nodes Involved:**
  - `When Called for Review`
  - `Compose Review Message`
  - `Check Audio Exists`
  - `Send Voice-over to Telegram`
  - `Notify Missing Voice`
  - `Attach Script File`
  - `Send Script to Telegram`
  - `Ask for Approval on Telegram`
  - `Check Approved`
  - `Mark Row Approved`
  - `Check Redo Requested`
  - `Requeue Topic`

- **Node Details:**
  - **When Called for Review**
    - *Type and Technical Role:* Execute Workflow Trigger node.
    - *Configuration Choices:* Receives item payloads passed from the parent execution loop via passthrough input source.
    - *Input/Output:* Entry point for sub-workflow executions; outputs to `Compose Review Message`.
  - **Compose Review Message**
    - *Type and Technical Role:* Code node formatting review text and text file attachments.
    - *Input/Output:* Input from trigger; outputs to `Check Audio Exists`.
  - **Check Audio Exists**
    - *Type and Technical Role:* If branching node.
    - *Configuration Choices:* Evaluates `has_audio` state.
    - *Input/Output:* Input from `Compose Review Message`. True branch goes to `Send Voice-over to Telegram`; False branch goes to `Notify Missing Voice`.
  - **Send Voice-over/Script to Telegram** (`Send Voice-over to Telegram`, `Notify Missing Voice`, `Attach Script File`, `Send Script to Telegram`)
    - *Type and Technical Role:* Telegram integration nodes (`sendDocument` and message dispatch operations).
    - *Configuration Choices:* Uploads binary WAV documents, formatted script text documents, and caption metadata blocks to the target chat ID.
    - *Input/Output:* Interconnected sequentially; terminate at `Ask for Approval on Telegram`.
    - *Failure Types:* Telegram file size limits (20MB+ audio files) or network disconnects.
  - **Ask for Approval on Telegram**
    - *Type and Technical Role:* Telegram Send and Wait node with approval buttons.
    - *Configuration Choices:* Configured with an interactive approval type (`double`), custom labels (`✅ Use it`, `🔁 Redo`), specified approver user IDs, and a wait duration derived from `wait_hours`.
    - *Input/Output:* Input from `Send Script to Telegram`; outputs to `Check Approved`.
    - *Failure Types:* Webhook reachability failures preventing button callbacks from resuming execution.
  - **Check Approved**
    - *Type and Technical Role:* If branching node.
    - *Configuration Choices:* Evaluates whether `$json.data?.approved` is true.
    - *Input/Output:* Input from `Ask for Approval on Telegram`. True branch goes to `Mark Row Approved`; False branch goes to `Check Redo Requested`.
  - **Mark Row Approved**
    - *Type and Technical Role:* n8n Data Table row update node.
    - *Configuration Choices:* Sets row status to `approved`, updates timestamps, and records the Telegram file ID.
    - *Input/Output:* Input from `Check Approved` (true branch); terminal node.
  - **Check Redo Requested**
    - *Type and Technical Role:* If branching node.
    - *Configuration Choices:* Evaluates whether `$json.data?.approved` is explicitly false.
    - *Input/Output:* Input from `Check Approved` (false branch); outputs to `Requeue Topic`.
  - **Requeue Topic**
    - *Type and Technical Role:* n8n Data Table row update node.
    - *Configuration Choices:* Resets row status to `queued` with notes indicating a requested redo.
    - *Input/Output:* Input from `Check Redo Requested`; terminal node.

---

#### Block 1.8: Topic Ingestion Form
**Overview:**  
Provides a public web form interface allowing users to submit new topic batches with specified niches, formats, languages, and scheduling dates.

- **Nodes Involved:**
  - `Add Topics Form`
  - `Create Queue Table for Form`
  - `Split Topics into Rows`
  - `Insert Topics into Queue`

- **Node Details:**
  - **Add Topics Form**
    - *Type and Technical Role:* Form Trigger node.
    - *Configuration Choices:* Provides textarea fields for topics (one per line), dropdowns for niche, format, language, start date, and optional writer notes.
    - *Input/Output:* Webhook entry point; outputs to `Create Queue Table for Form`.
  - **Create Queue Table for Form**
    - *Type and Technical Role:* n8n Data Table initialization node.
    - *Configuration Choices:* Ensures `vs_queue` table exists for incoming submissions.
    - *Input/Output:* Input from form trigger; outputs to `Split Topics into Rows`.
  - **Split Topics into Rows**
    - *Type and Technical Role:* Code node processing form input blocks.
    - *Configuration Choices:* Splits multiline topic inputs, strips list markers, deduplicates entries up to 60 items, and calculates staggered daily publish dates.
    - *Input/Output:* Input from table creation; outputs to `Insert Topics into Queue`.
  - **Insert Topics into Queue**
    - *Type and Technical Role:* n8n Data Table row insertion node.
    - *Configuration Choices:* Inserts generated topic objects as individual rows with `queued` status.
    - *Input/Output:* Input from `Split Topics into Rows`; terminal node.
    - *Failure Types:* Database constraint violations or date format conversion failures.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Run Every Morning | n8n-nodes-base.scheduleTrigger | Triggers workflow execution daily at 7 AM. | None | Set Channel Settings | Faceless voice-overs in batch with Gemini TTS |
| Set Channel Settings | n8n-nodes-base.set | Defines global channel parameters, models, and toggles. | Run Every Morning | Create Queue Table If Missing, Get Next Queued Topics | Faceless voice-overs in batch with Gemini TTS |
| Create Queue Table If Missing | n8n-nodes-base.dataTable | Ensures the `vs_queue` data table schema exists. | Set Channel Settings | None | 1 · Take the next topics |
| Get Next Queued Topics | n8n-nodes-base.dataTable | Fetches pending queued topics up to batch size limit. | Set Channel Settings | Check Queue Has Topics | 1 · Take the next topics |
| Check Queue Has Topics | n8n-nodes-base.if | Verifies if retrieved topics list is populated. | Get Next Queued Topics | Build Voice Direction Prompt, Notify Empty Queue | 1 · Take the next topics / Never silent |
| Notify Empty Queue | n8n-nodes-base.telegram | Sends an alert message if the topic queue is empty. | Check Queue Has Topics | None | Never silent |
| Build Voice Direction Prompt | n8n-nodes-base.code | Compiles niche prompts, styles, and format instructions. | Check Queue Has Topics | Write Script with Gemini | 2 · Write the script |
| Write Script with Gemini | @n8n/n8n-nodes-langchain.googleGemini | Generates structured JSON scripts using primary Gemini model. | Build Voice Direction Prompt | Check Script Quality, Write Script with Backup Gemini | 2 · Write the script |
| Write Script with Backup Gemini | @n8n/n8n-nodes-langchain.googleGemini | Fallback script generation using secondary Gemini model. | Write Script with Gemini | Check Script Quality | 2 · Write the script |
| Check Script Quality | n8n-nodes-base.code | Evaluates word counts, hooks, cliches, and scene structures. | Write Script with Gemini, Write Script with Backup Gemini | Check Script Exists | 3 · Check the script |
| Check Script Exists | n8n-nodes-base.if | Checks whether a valid script object was returned. | Check Script Quality | Check Fix Needed, List Topics Without Script | 3 · Check the script / Never silent |
| List Topics Without Script | n8n-nodes-base.code | Aggregates titles of topics that failed script generation. | Check Script Exists | Notify Missing Scripts | Never silent |
| Notify Missing Scripts | n8n-nodes-base.telegram | Sends warning notifications for failed script generations. | List Topics Without Script | None | Never silent |
| Check Fix Needed | n8n-nodes-base.if | Determines if script validation problems require correction. | Check Script Exists | Fix Script with Gemini, Pick Best Script Version | 3 · Check the script |
| Fix Script with Gemini | @n8n/n8n-nodes-langchain.googleGemini | Attempts to repair script issues using a low-temperature model. | Check Fix Needed | Pick Best Script Version | 3 · Check the script |
| Pick Best Script Version | n8n-nodes-base.code | Compares pre- and post-fix scripts and selects the best version. | Check Fix Needed, Fix Script with Gemini | Generate Voice with Gemini TTS | 3 · Check the script |
| Generate Voice with Gemini TTS | n8n-nodes-base.httpRequest | Requests WAV voice-over generation from Gemini TTS endpoints. | Pick Best Script Version | Build Voice-over Package, Generate Voice with Backup TTS | 4 · Voice it |
| Generate Voice with Backup TTS | n8n-nodes-base.httpRequest | Fallback voice generation using secondary TTS model. | Generate Voice with Gemini TTS | Build Voice-over Package | 4 · Voice it |
| Build Voice-over Package | n8n-nodes-base.code | Analyzes WAV buffers, calculates pause timings, and builds SRT. | Generate Voice with Gemini TTS, Generate Voice with Backup TTS | Check Listen-back Enabled | 4 · Voice it |
| Check Listen-back Enabled | n8n-nodes-base.if | Checks if audio exists and listen-back quality assurance is active. | Build Voice-over Package | Transcribe Voice with Gemini, Compare Transcript with Script | 5 · Listen back |
| Transcribe Voice with Gemini | @n8n/n8n-nodes-langchain.googleGemini | Transcribes generated audio files back into text for QA. | Check Listen-back Enabled | Compare Transcript with Script | 5 · Listen back |
| Compare Transcript with Script | n8n-nodes-base.code | Computes word-level match accuracy between script and transcript. | Check Listen-back Enabled, Transcribe Voice with Gemini | Save Results to Queue, Send Each Voice-over to Review | 6 · Save and hand off |
| Save Results to Queue | n8n-nodes-base.dataTable | Persists final package metadata and status to the data table. | Compare Transcript with Script | None | 6 · Save and hand off |
| Send Each Voice-over to Review | n8n-nodes-base.executeWorkflow | Invokes a sub-workflow run for individual review processing. | Compare Transcript with Script | None | 6 · Save and hand off / 7 · Review on Telegram |
| Add Topics Form | n8n-nodes-base.formTrigger | Webhook form trigger for submitting new topic batches. | None | Create Queue Table for Form | Add topics |
| Create Queue Table for Form | n8n-nodes-base.dataTable | Ensures data table exists when handling form submissions. | Add Topics Form | Split Topics into Rows | Add topics |
| Split Topics into Rows | n8n-nodes-base.code | Splits form textareas into individual structured topic items. | Create Queue Table for Form | Insert Topics into Queue | Add topics |
| Insert Topics into Queue | n8n-nodes-base.dataTable | Inserts newly formatted topic rows into the queue table. | Split Topics into Rows | None | Add topics |
| When Called for Review | n8n-nodes-base.executeWorkflowTrigger | Entry trigger for sub-workflow item review executions. | None | Compose Review Message | 7 · Review on Telegram |
| Compose Review Message | n8n-nodes-base.code | Formats Telegram review captions, assets, and text files. | When Called for Review | Check Audio Exists | 7 · Review on Telegram |
| Check Audio Exists | n8n-nodes-base.if | Verifies if valid voice audio was successfully synthesized. | Compose Review Message | Send Voice-over to Telegram, Notify Missing Voice | 7 · Review on Telegram |
| Send Voice-over to Telegram | n8n-nodes-base.telegram | Sends generated WAV audio documents to the Telegram channel. | Check Audio Exists | Attach Script File | 7 · Review on Telegram |
| Notify Missing Voice | n8n-nodes-base.telegram | Sends text alerts when voice synthesis fails completely. | Check Audio Exists | Attach Script File | 7 · Review on Telegram |
| Attach Script File | n8n-nodes-base.code | Prepares script text file binary payload for Telegram. | Send Voice-over to Telegram, Notify Missing Voice | Send Script to Telegram | 7 · Review on Telegram |
| Send Script to Telegram | n8n-nodes-base.telegram | Sends formatted script text documents to the Telegram channel. | Attach Script File | Ask for Approval on Telegram | 7 · Review on Telegram |
| Ask for Approval on Telegram | n8n-nodes-base.telegram | Sends interactive approval buttons and waits for user response. | Send Script to Telegram | Check Approved | 7 · Review on Telegram |
| Check Approved | n8n-nodes-base.if | Evaluates whether the user clicked the approve button. | Ask for Approval on Telegram | Mark Row Approved, Check Redo Requested | 7 · Review on Telegram |
| Mark Row Approved | n8n-nodes-base.dataTable | Updates row status to approved in the queue table. | Check Approved | None | 7 · Review on Telegram |
| Check Redo Requested | n8n-nodes-base.if | Evaluates whether the user requested a redo. | Check Approved | Requeue Topic | 7 · Review on Telegram |
| Requeue Topic | n8n-nodes-base.dataTable | Resets row status to queued for regeneration. | Check Redo Requested | None | 7 · Review on Telegram |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Main Trigger and Channel Setup**
   - Add a **Schedule Trigger** (`Run Every Morning`) configured to run daily at hour `7`.
   - Add a **Set (Edit Fields)** node (`Set Channel Settings`). Configure fields: `channel_name` (string), `channel_angle` (string), `telegram_chat_id` (string), `batch_size` (number: `3`), `default_language` (`en`), `listen_back_check` (boolean: `true`), `suno_music` (boolean: `true`), `tts_model` (`gemini-3.8-flash-tts`), `tts_backup_model` (`gemini-3.8-flash-lite-tts`), `approver_user_id` (string), and `wait_hours` (number: `48`).

2. **Set up Data Tables**
   - Add a **Data Table** node (`Create Queue Table If Missing`) set to resource `table`, operation `create`, table name `vs_queue`, with columns: `topic` (string), `niche` (string), `format` (string), `language` (string), `voice` (string), `notes` (string), `publish_date` (date), `status` (string), `title` (string), `words` (number), `duration_s` (number), `engine` (string), `checks` (string), `listen_back` (number), `facts_to_verify` (string), `suno_prompt` (string), `package_json` (string), `audio_file_id` (string), and `updated` (date). Enable `createIfNotExists`.
   - Add a second **Data Table** node (`Get Next Queued Topics`) set to resource `row`, operation `get`, filtering where `status` equals `queued`, ordered by `publish_date` ASC, limited by `={{ $('Set Channel Settings').first().json.batch_size }}`.
   - Connect `Set Channel Settings` to both table nodes.

3. **Configure Queue Validation and Prompt Compilation**
   - Add an **If** node (`Check Queue Has Topics`) checking `={{ $json.topic }}` is not empty. Connect `Get Next Queued Topics` here.
   - Connect the false branch to a **Telegram** node (`Notify Empty Queue`) sending an empty queue warning.
   - Connect the true branch to a **Code** node (`Build Voice Direction Prompt`) containing the formatting libraries, format rules, niches, cliches, and template string replacement logic provided in the reference JSON.

4. **Implement Script Generation and Validation**
   - Add a **Google Gemini Chat Model** node (`Write Scriptwith Gemini`) using model ID `models/gemini-flash-lite-latest`, temperature `0.9`, system message `={{ $('Build Voice Direction Prompt').item.json.system }}`, and text message content `={{ $('Build Voice Direction Prompt').item.json.prompt }}` with JSON output enabled. Configure error output routing and 3 max retries.
   - Add a backup **Google Gemini Chat Model** node (`Write Script with Backup Gemini`) using model ID `models/gemini-3.1-flash-lite`, temperature `0.9`, and identical system/message bindings. Connect error output from primary Gemini here.
   - Add a **Code** node (`Check Script Quality`) to sanitize text, check word counts, validate hooks, and build validation problem arrays. Connect both Gemini writing nodes to this node.
   - Add an **If** node (`Check Script Exists`) evaluating `={{ $json.has_script }}`.
   - Connect the false branch to a **Code** node (`List Topics Without Script`) and a **Telegram** node (`Notify Missing Scripts`).

5. **Build Script Repair Logic**
   - Add an **If** node (`Check Fix Needed`) checking if `={{ $json.problems.length }}` > `0`. Connect true branch from `Check Script Exists` here.
   - Add a **Google Gemini Chat Model** node (`Fix Script with Gemini`) using model ID `models/gemini-3.1-flash-lite`, temperature `0.4`, system message bindings, and fix prompt content.
   - Add a **Code** node (`Pick Best Script Version`) receiving inputs from `Check Fix Needed` (false branch) and `Fix Script with Gemini`. Compares problem lengths and constructs TTS bodies and Suno blocks.

6. **Implement TTS Voice Synthesis and Timing Analysis**
   - Add an **HTTP Request** node (`Generate Voice with Gemini TTS`) making a `POST` request to `https://generativelanguage.googleapis.com/v1beta/interactions`, using pre-defined **Google Palm API** credentials with a 600,000ms timeout and JSON body `={{ JSON.stringify($json.tts) }}`. Configure error output routing and 2 retries.
   - Add a backup **HTTP Request** node (`Generate Voice with Backup TTS`) using identical parameters except substituting the model with `={{ $('Set Channel Settings').first().json.tts_backup_model }}`. Connect error output from primary TTS here.
   - Add a **Code** node (`Build Voice-over Package`) parsing base64 audio responses, analyzing WAV RMS levels for pauses, timing scenes, building SRT captions, and outputting binary WAV attachments. Connect both TTS request nodes here.

7. **Configure Transcription QA and Database Persistence**
   - Add an **If** node (`Check Listen-back Enabled`) verifying `={{ $json.has_audio }}` is true and `={{ $('Set Channel Settings').first().json.listen_back_check }}` is true.
   - Add a **Google Gemini Audio Analysis** node (`Transcribe Voice with Gemini`) using model ID `models/gemini-3.1-flash-lite`, resource `audio`, operation `analyze`, binary property `data`, and transcription instruction text. Connect true branch here.
   - Add a **Code** node (`Compare Transcript with Script`) executing Levenshtein distance calculations between script plain text and transcribed text. Connect both `Check Listen-back Enabled` (false branch) and `Transcribe Voice with Gemini` here.
   - Add a **Data Table** node (`Save Results to Queue`) set to resource `row`, operation `update`, updating row where `id` equals `={{ $json.row_id }}` with execution metrics, status (`review`), listen-back scores, and serialized package JSON.
   - Add an **Execute Workflow** node (`Send Each Voice-over to Review`) set to mode `each`, workflow ID `={{ $workflow.id }}`, with `waitForSubWorkflow: false`. Connect both database save and workflow invocation nodes to `Compare Transcript with Script`.

8. **Implement Topic Ingestion Form**
   - Add a **Form Trigger** node (`Add Topics Form`) with fields: Textarea (`Topics (one per line)`), Dropdown (`Niche`), Dropdown (`Format`), Dropdown (`Language`), Date (`First publish date`), and Text (`Notes for the writer`).
   - Add a **Data Table** node (`Create Queue Table for Form`) ensuring table schema exists.
   - Add a **Code** node (`Split Topics into Rows`) parsing multiline inputs into individual daily queued records.
   - Add a **Data Table** node (`Insert Topics into Queue`) set to resource `row`, operation `insert`, mapping topic fields into `vs_queue`. Connect sequentially.

9. **Build Telegram Review Sub-Workflow**
   - Add an **Execute Workflow Trigger** node (`When Called for Review`) set to passthrough input source.
   - Add a **Code** node (`Compose Review Message`) formatting message headers, check summaries, listen-back metrics, and text file attachments.
   - Add an **If** node (`Check Audio Exists`) evaluating `={{ $json.has_audio }}`.
   - Add **Telegram** nodes (`Send Voice-over to Telegram` using `sendDocument` operation, and `Notify Missing Voice` for text alerts).
   - Add a **Code** node (`Attach Script File`) mapping document file IDs and script binaries.
   - Add a **Telegram** node (`Send Script to Telegram`) sending text document attachments.
   - Add an **Interactive Approval Telegram** node (`Ask for Approval on Telegram`) configured with `chatApproval: true`, response type `approval`, double approval options (`✅ Use it`, `🔁 Redo`), approver IDs, and a wait interval using `={{ $('When Called for Review').first().json.review.wait_hours }}` hours.
   - Add an **If** node (`Check Approved`) verifying `={{ $json.data?.approved }}`.
   - Connect the true branch to a **Data Table** node (`Mark Row Approved`) updating status to `approved`.
   - Connect the false branch to an **If** node (`Check Redo Requested`) checking `={{ $json.data?.approved === false }}`.
   - Connect the true branch of the redo check to a **Data Table** node (`Requeue Topic`) setting status back to `queued`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Faceless voice-overs in batch with Gemini TTS · Made by Amelqa | [Amelqa Official Website](https://amelqa.com) |
| Free Google Gemini (PaLM) API Key Generation | [Google AI Studio](https://aistudio.google.com) |
| Telegram Bot Token Generation via BotFather | [@BotFather](https://t.me/BotFather) |
| Telegram Chat ID Retrieval Bot | [@userinfobot](https://t.me/userinfobot) |
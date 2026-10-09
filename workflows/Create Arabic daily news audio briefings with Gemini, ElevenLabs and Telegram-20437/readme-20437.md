Create Arabic daily news audio briefings with Gemini, ElevenLabs and Telegram

https://n8nworkflows.xyz/workflows/create-arabic-daily-news-audio-briefings-with-gemini--elevenlabs-and-telegram-20437


# Create Arabic daily news audio briefings with Gemini, ElevenLabs and Telegram

### 1. Workflow Overview

This workflow automates the generation and multi-channel distribution of daily Arabic news briefings. Operating on a fixed daily schedule, it fetches recent news items from an RSS feed, cross-references them against a history log to prevent duplication, processes the content via Google Gemini to select and rewrite the top stories into Modern Standard Arabic, validates and self-heals the output if necessary, and finally broadcasts the output as both a formatted text message and an AI-generated audio voiceover to Telegram, while recording the execution metadata to Google Sheets.

The architecture is divided into three distinct functional blocks:
- **1.1 Schedule, Fetch, and Deduplication:** Triggers the automation daily at 07:00, pulling live RSS feed entries concurrently with historical tracking data from Google Sheets.
- **1.2 AI Ranking and Self-Healing JSON Validation:** Merges incoming data streams, prompts Google Gemini to curate and translate/rewrite stories while filtering out duplicate topics, and validates the returned payload via a custom script node with an integrated self-healing repair loop.
- **1.3 Delivery and Tracking:** Distributes the validated briefing simultaneously as a Telegram text message and an ElevenLabs-generated Arabic audio clip, concluding with an append operation to a Google Sheets tracking log.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Schedule, Fetch, and Deduplication
- **Overview:** This block initializes the workflow execution on a daily schedule and concurrently retrieves fresh RSS news articles alongside historical logs of previously published headlines to enforce deduplication.
- **Nodes Involved:** 
  - `Run Every Morning at 7 AM`
  - `Fetch Latest Headlines`
  - `Load Previously Sent Headlines`
  - `Combine News and History`
- **Node Details:**
  - **Run Every Morning at 7 AM**
    - *Type and Technical Role:* `n8n-nodes-base.scheduleTrigger` (v1.2). Acts as the cron-based time trigger.
    - *Configuration Choices:* Configured with a cron expression (`0 7 * * *`) to execute daily at 07:00.
    - *Key Expressions:* None.
    - *Input/Output:* No inputs; outputs to `Fetch Latest Headlines` and `Load Previously Sent Headlines`.
    - *Version-specific Requirements:* Version 1.2.
    - *Edge Cases / Failure Types:* Execution might be missed if the n8n instance is offline at the scheduled time; depends on the host system clock/timezone.
  - **Fetch Latest Headlines**
    - *Type and Technical Role:* `n8n-nodes-base.rssFeedRead` (v1.2). Reads and parses an external RSS feed.
    - *Configuration Choices:* Target URL set to Al Jazeera Arabic RSS feed (`https://www.aljazeera.net/aljazeerarss/a7c186be-1a50-4673-baea-1588d8f9a584`) with a limit of 20 items.
    - *Key Expressions:* None.
    - *Input/Output:* Input from `Run Every Morning at 7 AM`; output connected to `Combine News and History` (Input 1).
    - *Version-specific Requirements:* Version 1.2.
    - *Edge Cases / Failure Types:* Network timeouts, invalid XML/RSS structure, or feed unavailability.
  - **Load Previously Sent Headlines**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (v4.5). Reads tracking rows from a specified spreadsheet.
    - *Configuration Choices:* Resource set to `sheetWithinDocument`, operation set to `read`. Sheet name specified as `Briefings` with document ID `YOUR_GOOGLE_SHEET_ID`. Returns all rows with the header row at index 1.
    - *Key Expressions:* None.
    - *Input/Output:* Input from `Run Every Morning at 7 AM`; output connected to `Combine News and History` (Input 2).
    - *Version-specific Requirements:* Version 4.5.
    - *Edge Cases / Failure Types:* Google Sheets API authentication failure, missing sheet, or incorrect document ID.
  - **Combine News and History**
    - *Type and Technical Role:* `n8n-nodes-base.merge` (v3). Combines multiple incoming data streams into a single dataset.
    - *Configuration Choices:* Mode set to `combineByPosition` with clash handling set to `preferInput1`.
    - *Key Expressions:* None.
    - *Input/Output:* Inputs from `Fetch Latest Headlines` (Input 1) and `Load Previously Sent Headlines` (Input 2); output connected to `Rank and Rewrite 5 Fresh Stories`.
    - *Version-specific Requirements:* Version 3.
    - *Edge Cases / Failure Types:* Misaligned payload structures if input data sizes differ significantly.

---

#### Block 1.2: AI Ranking and Self-Healing JSON Validation
- **Overview:** This block processes the combined news and historical exclusion list using Google Gemini to produce a structured Modern Standard Arabic briefing. A code node validates the output, triggering a secondary AI repair loop if the payload is malformed.
- **Nodes Involved:**
  - `Rank and Rewrite 5 Fresh Stories`
  - `Validate and Detect Breaking News`
  - `Is Briefing Valid?`
  - `Repair Malformed JSON Output`
- **Node Details:**
  - **Rank and Rewrite 5 Fresh Stories**
    - *Type and Technical Role:* `n8n-nodes-base.googleGemini` (v1). Interfaces with the Google Gemini generative model API.
    - *Configuration Choices:* Model set to `models/gemini-2.5-flash`, temperature set to `0.4`, maximum output tokens set to `2048`.
    - *Key Expressions:* Uses dynamic prompt construction evaluating `{{ $now.format('yyyy-MM-dd') }}`, mapping items from `{{ JSON.stringify($('Fetch Latest Headlines').all().map(...)) }}`, and extracting historical exclusion filters from `{{ JSON.stringify($('Load Previously Sent Headlines').all().map(i => i.json.Headline)) }}`.
    - *Input/Output:* Input from `Combine News and History`; output connected to `Validate and Detect Breaking News`.
    - *Version-specific Requirements:* Version 1. Requires Google Gemini credentials.
    - *Edge Cases / Failure Types:* API rate limits, refusal due to safety filters, or invalid JSON string generation by the model.
  - **Validate and Detect Breaking News**
    - *Type and Technical Role:* `n8n-nodes-base.code` (v2). Executes custom JavaScript to parse, validate, and enrich the AI output.
    - *Configuration Choices:* Custom JS code cleans markdown code blocks, parses JSON, performs stop-word-based frequency analysis to detect breaking news topics (frequency ≥ 3), and formats the final Markdown briefing string.
    - *Key Expressions:* Parses incoming payloads from `$input.first().json.text` or `$input.first().json.output`.
    - *Input/Output:* Input from `Rank and Rewrite 5 Fresh Stories` (or `Repair Malformed JSON Output`); output connected to `Is Briefing Valid?`.
    - *Version-specific Requirements:* Version 2.
    - *Edge Cases / Failure Types:* Syntax exceptions during JSON parsing if the model output is heavily corrupted.
  - **Is Briefing Valid?**
    - *Type and Technical Role:* `n8n-nodes-base.if` (v2). Routes workflow execution based on conditional evaluation.
    - *Configuration Choices:* Evaluates whether `{{ $json.valid }}` equals `true`.
    - *Key Expressions:* `={{ $json.valid }}`.
    - *Input/Output:* Input from `Validate and Detect Breaking News`; output index 0 connects to downstream delivery nodes, output index 1 connects to `Repair Malformed JSON Output`.
    - *Version-specific Requirements:* Version 2.
    - *Edge Cases / Failure Types:* Boolean evaluation mismatch if the validation property is undefined.
  - **Repair Malformed JSON Output**
    - *Type and Technical Role:* `n8n-nodes-base.googleGemini` (v1). Secondary AI node dedicated to error correction.
    - *Configuration Choices:* Model set to `models/gemini-2.5-flash`, temperature set to `0.1`, max output tokens set to `2048`.
    - *Key Expressions:* Passes the broken payload via `{{ $json.raw }}` to request a clean JSON array structure.
    - *Input/Output:* Input from `Is Briefing Valid?` (false branch); output loops back to `Validate and Detect Breaking News`.
    - *Version-specific Requirements:* Version 1. Requires Google Gemini credentials.
    - *Edge Cases / Failure Types:* Repeated generation of malformed output if the broken text cannot be salvaged.

---

#### Block 1.3: Delivery and Tracking
- **Overview:** This block formats and distributes the verified news briefing to Telegram as both a formatted text message and an ElevenLabs-synthesized audio voiceover file, concluding by appending a delivery log entry to Google Sheets.
- **Nodes Involved:**
  - `Send Telegram Text Briefing`
  - `Log Briefing to Tracking Sheet`
  - `Generate Arabic Voiceover`
  - `Send Telegram Audio Briefing`
- **Node Details:**
  - **Send Telegram Text Briefing**
    - *Type and Technical Role:* `n8n-nodes-base.telegram` (v1.2). Sends a message via the Telegram Bot API.
    - *Configuration Choices:* Chat ID set to `123456789`, additional fields configured with `parse_mode: "Markdown"` and `disable_web_page_preview: true`.
    - *Key Expressions:* Text content populated via `={{ $json.briefing }}`.
    - *Input/Output:* Input from `Is Briefing Valid?` (true branch); no downstream connections.
    - *Version-specific Requirements:* Version 1.2. Requires Telegram Bot credentials.
    - *Edge Cases / Failure Types:* Invalid chat ID, markdown parsing errors due to unescaped characters, or bot blocking restrictions.
  - **Log Briefing to Tracking Sheet**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (v4.5). Appends a new row of data to a spreadsheet.
    - *Configuration Choices:* Resource set to `sheetWithinDocument`, operation set to `append`. Mapped columns: `Date` (`={{ $json.date }}`), `Status` (`Delivered`), `Summary` (`={{ $json.summary }}`), `Headline` (`={{ $json.headline }}`), `Audio Sent` (`Yes`), `Story Count` (`={{ $json.storyCount }}`). Document ID set to `YOUR_GOOGLE_SHEET_ID` on sheet `Briefings`.
    - *Key Expressions:* Dynamic column mapping expressions referencing workflow JSON properties.
    - *Input/Output:* Input from `Is Briefing Valid?` (true branch); no downstream connections.
    - *Version-specific Requirements:* Version 4.5. Requires Google Sheets credentials.
    - *Edge Cases / Failure Types:* Schema mismatches, missing columns, or API quota limits.
  - **Generate Arabic Voiceover**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (v4.2). Performs an HTTP POST request to an external API.
    - *Configuration Choices:* Method set to `POST`, URL set to `https://api.elevenlabs.io/v1/text-to-speech/pNInz6obpgDQGcFmaJgB`, content type `json`, response format set to `file`. Request body cleans markdown and symbols from the briefing text and targets model `eleven_multilingual_v2` with stability `0.5` and similarity boost `0.75`.
    - *Key Expressions:* Request body constructed using `={{ JSON.stringify($json.briefing.replace(/\*\*/g, '').replace(/[🔹📰]/g, '')) }}`.
    - *Input/Output:* Input from `Is Briefing Valid?` (true branch); output connected to `Send Telegram Audio Briefing`.
    - *Version-specific Requirements:* Version 4.2. Requires ElevenLabs API credentials.
    - *Edge Cases / Failure Types:* API authentication errors, character limit overages, or timeout on large payloads.
  - **Send Telegram Audio Briefing**
    - *Type and Technical Role:* `n8n-nodes-base.telegram` (v1.2). Sends a binary media file (audio) via the Telegram Bot API.
    - *Configuration Choices:* Chat ID set to `123456789`, additional fields configured with caption `={{ 'Audio briefing for ' + $json.date }}`.
    - *Key Expressions:* Caption string `={{ 'Audio briefing for ' + $json.date }}`.
    - *Input/Output:* Input from `Generate Arabic Voiceover`; no downstream connections.
    - *Version-specific Requirements:* Version 1.2. Requires Telegram Bot credentials.
    - *Edge Cases / Failure Types:* Missing binary file input, unsupported audio format, or file size exceeding Telegram Bot API upload limits.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Run Every Morning at 7 AM | n8n-nodes-base.scheduleTrigger | Triggers workflow execution daily at 07:00 | None | Fetch Latest Headlines, Load Previously Sent Headlines | Create Arabic daily news audio briefings with Gemini, ElevenLabs, and Telegram<br><br>Schedule, fetch, and deduplicate |
| Fetch Latest Headlines | n8n-nodes-base.rssFeedRead | Fetches RSS news entries from Al Jazeera | Run Every Morning at 7 AM | Combine News and History | Create Arabic daily news audio briefings with Gemini, ElevenLabs, and Telegram<br><br>Schedule, fetch, and deduplicate |
| Load Previously Sent Headlines | n8n-nodes-base.googleSheets | Reads historical log entries from Google Sheets | Run Every Morning at 7 AM | Combine News and History | Create Arabic daily news audio briefings with Gemini, ElevenLabs, and Telegram<br><br>Schedule, fetch, and deduplicate |
| Combine News and History | n8n-nodes-base.merge | Merges feed items and historical data by position | Fetch Latest Headlines, Load Previously Sent Headlines | Rank and Rewrite 5 Fresh Stories | Create Arabic daily news audio briefings with Gemini, ElevenLabs, and Telegram<br><br>Schedule, fetch, and deduplicate |
| Rank and Rewrite 5 Fresh Stories | n8n-nodes-base.googleGemini | AI curation and Modern Standard Arabic rewriting | Combine News and History | Validate and Detect Breaking News | Create Arabic daily news audio briefings with Gemini, ElevenLabs, and Telegram<br><br>AI ranking and self-healing JSON |
| Validate and Detect Breaking News | n8n-nodes-base.code | Parses JSON, detects breaking news, formats briefing | Rank and Rewrite 5 Fresh Stories (or Repair Malformed JSON Output) | Is Briefing Valid? | Create Arabic daily news audio briefings with Gemini, ElevenLabs, and Telegram<br><br>AI ranking and self-healing JSON |
| Is Briefing Valid? | n8n-nodes-base.if | Evaluates validation status of processed stories | Validate and Detect Breaking News | Send Telegram Text Briefing, Log Briefing to Tracking Sheet, Generate Arabic Voiceover (or Repair Malformed JSON Output) | Create Arabic daily news audio briefings with Gemini, ElevenLabs, and Telegram<br><br>AI ranking and self-healing JSON |
| Repair Malformed JSON Output | n8n-nodes-base.googleGemini | Self-healing fallback call to fix broken JSON | Is Briefing Valid? | Validate and Detect Breaking News | Create Arabic daily news audio briefings with Gemini, ElevenLabs, and Telegram<br><br>AI ranking and self-healing JSON |
| Send Telegram Text Briefing | n8n-nodes-base.telegram | Broadcasts formatted text briefing to Telegram | Is Briefing Valid? | None | Create Arabic daily news audio briefings with Gemini, ElevenLabs, and Telegram<br><br>Deliver text, audio, and log |
| Log Briefing to Tracking Sheet | n8n-nodes-base.googleSheets | Appends delivery log entry to Google Sheets | Is Briefing Valid? | None | Create Arabic daily news audio briefings with Gemini, ElevenLabs, and Telegram<br><br>Deliver text, audio, and log |
| Generate Arabic Voiceover | n8n-nodes-base.httpRequest | Calls ElevenLabs TTS API to generate audio file | Is Briefing Valid? | Send Telegram Audio Briefing | Create Arabic daily news audio briefings with Gemini, ElevenLabs, and Telegram<br><br>Deliver text, audio, and log |
| Send Telegram Audio Briefing | n8n-nodes-base.telegram | Sends generated audio voiceover to Telegram | Generate Arabic Voiceover | None | Create Arabic daily news audio briefings with Gemini, ElevenLabs, and Telegram<br><br>Deliver text, audio, and log |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Schedule Trigger Node:**
   - Add a **Schedule Trigger** node (`n8n-nodes-base.scheduleTrigger`).
   - Set interval rule to Cron expression: `0 7 * * *`.
   - Name it `Run Every Morning at 7 AM`.

2. **Create the RSS Feed Reader Node:**
   - Add an **RSS Feed Read** node (`n8n-nodes-base.rssFeedRead`).
   - Set URL to `https://www.aljazeera.net/aljazeerarss/a7c186be-1a50-4673-baea-1588d8f9a584`.
   - Set options limit to `20`.
   - Name it `Fetch Latest Headlines`. Connect output from `Run Every Morning at 7 AM`.

3. **Create the Google Sheets Reader Node:**
   - Add a **Google Sheets** node (`n8n-nodes-base.googleSheets`).
   - Set resource to `sheetWithinDocument` and operation to `read`.
   - Configure Document ID to `YOUR_GOOGLE_SHEET_ID` and Sheet Name to `Briefings` (header row 1, first data row 2).
   - Name it `Load Previously Sent Headlines`. Connect output from `Run Every Morning at 7 AM`.
   - *Credential Requirement:* Connect valid Google Sheets OAuth2/Service Account credentials.

4. **Create the Merge Node:**
   - Add a **Merge** node (`n8n-nodes-base.merge`).
   - Set mode to `kombineByPosition` / `combineByPosition` with clash handling set to `preferInput1`.
   - Name it `Combine News and History`. Connect Input 1 from `Fetch Latest Headlines` and Input 2 from `Load Previously Sent Headlines`.

5. **Create the Primary Gemini Node:**
   - Add a **Google Gemini** node (`n8n-nodes-base.googleGemini`).
   - Set model to `models/gemini-2.5-flash`, temperature to `0.4`, and max output tokens to `2048`.
   - Set prompt to accept the dynamic JavaScript template evaluating fresh items and historical exclusion arrays.
   - Name it `Rank and Rewrite 5 Fresh Stories`. Connect input from `Combine News and History`.
   - *Credential Requirement:* Connect Google Gemini API Key credentials.

6. **Create the Validation and Breaking News Code Node:**
   - Add a **Code** node (`n8n-nodes-base.code`).
   - Paste the JavaScript snippet that cleans markdown blocks, parses JSON, checks frequency for breaking topics, and formats the markdown briefing string.
   - Name it `Validate and Detect Breaking News`. Connect input from `Rank and Rewrite 5 Fresh Stories`.

7. **Create the IF Condition Node:**
   - Add an **If** node (`n8n-nodes-base.if`).
   - Set condition to evaluate whether `{{ $json.valid }}` equals `true`.
   - Name it `Is Briefing Valid?`. Connect input from `Validate and Detect Breaking News`.

8. **Create the Repair Gemini Node (Self-Healing Loop):**
   - Add a **Google Gemini** node (`n8n-nodes-base.googleGemini`).
   - Set model to `models/gemini-2.5-flash`, temperature to `0.1`, max output tokens to `2048`.
   - Set prompt to reconstruct valid JSON from broken output `{{ $json.raw }}`.
   - Name it `Repair Malformed JSON Output`. Connect input from `Is Briefing Valid?` (false branch, index 1) and output back to `Validate and Detect Breaking News`.
   - *Credential Requirement:* Connect Google Gemini API Key credentials.

9. **Create the Telegram Text Node:**
   - Add a **Telegram** node (`n8n-nodes-base.telegram`).
   - Set chat ID to `123456789`, text to `={{ $json.briefing }}`, parse mode to `Markdown`, and disable web page preview to `true`.
   - Name it `Send Telegram Text Briefing`. Connect input from `Is Briefing Valid?` (true branch, index 0).
   - *Credential Requirement:* Connect Telegram Bot API credentials.

10. **Create the Google Sheets Append Node:**
    - Add a **Google Sheets** node (`n8n-nodes-base.googleSheets`).
    - Set resource to `sheetWithinDocument`, operation to `append`.
    - Map columns: `Date` (`={{ $json.date }}`), `Status` (`Delivered`), `Summary` (`={{ $json.summary }}`), `Headline` (`={{ $json.headline }}`), `Audio Sent` (`Yes`), `Story Count` (`={{ $json.storyCount }}`). Set Document ID to `YOUR_GOOGLE_SHEET_ID` and Sheet Name to `Briefings`.
    - Name it `Log Briefing to Tracking Sheet`. Connect input from `Is Briefing Valid?` (true branch, index 0).
    - *Credential Requirement:* Connect Google Sheets credentials.

11. **Create the ElevenLabs HTTP Request Node:**
    - Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`).
    - Set method to `POST`, URL to `https://api.elevenlabs.io/v1/text-to-speech/pNInz6obpgDQGcFmaJgB`, content type to `json`, and response format to `file`.
    - Set body to include cleaned text, model ID `eleven_multilingual_v2`, and voice settings.
    - Name it `Generate Arabic Voiceover`. Connect input from `Is Briefing Valid?` (true branch, index 0).
    - *Credential Requirement:* Configure HTTP Header Authentication with your ElevenLabs API Key.

12. **Create the Telegram Audio Node:**
    - Add a **Telegram** node (`n8n-nodes-base.telegram`).
    - Set chat ID to `123456789` and caption to `={{ 'Audio briefing for ' + $json.date }}`.
    - Name it `Send Telegram Audio Briefing`. Connect input from `Generate Arabic Voiceover`.
    - *Credential Requirement:* Connect Telegram Bot API credentials.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Create Arabic daily news audio briefings with Gemini, ElevenLabs, and Telegram | Workflow title and functional overview |
| Required Google Sheets Columns | `Date`, `Headline`, `Summary`, `Story Count`, `Audio Sent`, `Status` |
Mine YouTube competitor content ideas with Google Gemini and Google Sheets

https://n8nworkflows.xyz/workflows/mine-youtube-competitor-content-ideas-with-google-gemini-and-google-sheets-19672


# Mine YouTube competitor content ideas with Google Gemini and Google Sheets

### 1. Workflow Overview

The **YouTube Competitor Idea Miner** workflow automates the discovery of new content ideas by monitoring competitor YouTube channels via RSS feeds. It runs either on a weekly schedule or via an interactive web form submission, utilizes Google Gemini to generate and score differentiated content angles based on competitor uploads, records processed history to Google Sheets, and emails a formatted digest of high-performing ideas.

The system logic is divided into the following functional blocks:
- **1.1 Input Reception & Configuration:** Captures execution triggers (schedule or form submission) and defines runtime variables such as channel lists, processing limits, score thresholds, and target Google Sheets parameters.
- **1.2 Seen State Retrieval:** Queries the Google Sheets database to pull existing processed video identifiers, establishing a state registry to prevent duplicate analysis.
- **1.3 RSS Ingestion & Parsing:** Splits the configured competitor channel IDs, iterates through each channel via an HTTP request to its RSS feed, and extracts structured video records.
- **1.4 Candidate Queue Construction:** Merges all harvested videos, filters out items present in the seen registry, injects optional test fixtures, and determines if valid candidate items exist.
- **1.5 AI Generation & Processing:** Iterates through candidate videos, executes a LangChain model call using Google Gemini to produce structured JSON content ideas, and sanitizes the output.
- **1.6 Evaluation, Logging & Persistence:** Assesses the AI-assigned opportunity score against a minimum threshold, appends qualified ideas to the "Video Ideas" sheet, and logs all processed videos into the "Seen Videos" sheet.
- **1.7 Digest Compilation & Notification:** Compiles a run summary of saved ideas, evaluates whether reporting criteria are met, and dispatches a summary email through Gmail.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Configuration
This block initiates the workflow from either an automated weekly schedule or a manual form interface, consolidating runtime configuration parameters into a unified data structure.

- **Weekly Channel Scan Trigger**
  - **Type & Role:** Schedule Trigger node (`n8n-nodes-base.scheduleTrigger`). Initiates automated scans every Monday at 08:00 AM.
  - **Configuration:** Interval set to weeks, triggered on day 1 at hour 8.
  - **Expressions/Variables:** None.
  - **Connections:** Output connects to `Configure Scan Parameters`.
  - **Edge Cases/Failures:** Missed executions if the n8n instance is offline during the scheduled window.

- **Form Submission Trigger**
  - **Type & Role:** Form Trigger node (`n8n-nodes-base.formTrigger`). Provides an interactive endpoint to manually input a comma-separated list of competitor YouTube channel IDs.
  - **Configuration:** Form title set to "Competitors Channel IDs" with a single required text field labeled "Channel IDs".
  - **Expressions/Variables:** None.
  - **Connections:** Output connects to `Configure Scan Parameters`.
  - **Edge Cases/Failures:** Webhook routing errors or submission validation failures if the field is left empty.

- **Configure Scan Parameters**
  - **Type & Role:** Set node (`n8n-nodes-base.set`). Establishes persistent variables for the workflow execution.
  - **Configuration:** Assigns runtime parameters: `competitorChannelIds`, `maxVideosPerChannel` (5), `maxCandidatesPerRun` (10), `ideaScoreThreshold` (7), `spreadsheetId`, `seenVideosSheetName` ("Seen Videos"), `ideasSheetName` ("Video Ideas"), `notifyEmail`, and `FORCE_CHANGE_FOR`.
  - **Expressions/Variables:** Uses `={{ $json['Channel IDs'] }}` to capture input from the Form Submission Trigger (falls back to undefined if triggered via schedule).
  - **Connections:** Input from `Weekly Channel Scan Trigger` or `Form Submission Trigger`; output connects to `Read Seen Video IDs from Sheet`.
  - **Edge Cases/Failures:** Missing channel ID properties when initiated via the weekly schedule if no default fallback is defined in the expression context.

---

#### 2.2 Seen State Retrieval
This block queries external storage to obtain a historical ledger of previously analyzed videos.

- **Read Seen Video IDs from Sheet**
  - **Type & Role:** Google Sheets node (`n8n-nodes-base.googleSheets`). Reads existing records from the target tracking sheet.
  - **Configuration:** Operation set to read/list rows with error handling configured to continue regular output upon failure.
  - **Expressions/Variables:** `documentId` uses `={{ $('Configure Scan Parameters').first().json.spreadsheetId }}`; `sheetName` uses `={{ $('Configure Scan Parameters').first().json.seenVideosSheetName }}`.
  - **Connections:** Input from `Configure Scan Parameters`; output connects to `Split Channel ID List`.
  - **Credentials:** Google Sheets OAuth2 API.
  - **Edge Cases/Failures:** Authentication expiration, invalid spreadsheet IDs, or missing sheet tabs resulting in empty sets (mitigated by `onError: continueRegularOutput`).

---

#### 2.3 RSS Ingestion & Parsing
This block processes the target list of competitor channels, querying individual RSS feeds and extracting structured video metadata.

- **Split Channel ID List**
  - **Type & Role:** Code node (`n8n-nodes-base.code`). Splits a comma-separated string of channel IDs into individual items.
  - **Configuration:** JavaScript parsing block splitting on commas, trimming whitespace, and filtering out empty values.
  - **Expressions/Variables:** Reads `competitorChannelIds` from `Configure Scan Parameters`.
  - **Connections:** Input from `Read Seen Video IDs from Sheet`; output connects to `Batch Process Channels`.
  - **Edge Cases/Failures:** Malformed strings resulting in zero valid channel tokens.

- **Batch Process Channels**
  - **Type & Role:** Split In Batches node (`n8n-nodes-base.splitInBatches`). Controls iteration over individual channel items.
  - **Configuration:** Default batch size settings.
  - **Expressions/Variables:** None.
  - **Connections:** Loop input from `Split Channel ID List` and `Parse Videos from RSS`; loop output branches to `Fetch YouTube Channel RSS` (looping) and `Combine Parsed Video Data` (completion).

- **Fetch YouTube Channel RSS**
  - **Type & Role:** HTTP Request node (`n8n-nodes-base.httpRequest`). Retrieves the raw Atom RSS XML feed for a given channel ID.
  - **Configuration:** GET request with custom User-Agent and Accept headers. Response format set to text with error tolerance enabled (`neverError: true`).
  - **Expressions/Variables:** URL uses `=https://www.youtube.com/feeds/videos.xml?channel_id={{ $json.channelId }}`.
  - **Connections:** Input from `Batch Process Channels`; output connects to `Parse Videos from RSS`.
  - **Edge Cases/Failures:** HTTP timeouts, rate limiting, or invalid channel IDs returning 404 (handled by `neverError`).

- **Parse Videos from RSS**
  - **Type & Role:** Code node (`n8n-nodes-base.code`). Parses raw XML feed strings into clean video objects.
  - **Configuration:** JavaScript regex-based extraction parsing `<entry>`, `<yt:videoId>`, `<title>`, `<published>`, and `<link>` tags, slicing results to `maxVideosPerChannel`.
  - **Expressions/Variables:** References configuration parameters and item payloads.
  - **Connections:** Input from `Fetch YouTube Channel RSS`; output connects to `Batch Process Channels`.
  - **Edge Cases/Failures:** Changes to YouTube's RSS XML schema breaking regex matches, yielding empty arrays.

---

#### 2.4 Candidate Queue Construction
This block aggregates all parsed channel videos, eliminates duplicates using the seen state registry, and applies operational limits.

- **Combine Parsed Video Data**
  - **Type & Role:** Code node (`n8n-nodes-base.code`). Collects all video items generated across asynchronous channel iteration loops.
  - **Configuration:** Uses `.all()` method on the upstream parsing node to gather all batched iterations.
  - **Connections:** Input from `Batch Process Channels`; output connects to `Deduplicate and Queue Candidates`.

- **Deduplicate and Queue Candidates**
  - **Type & Role:** Code node (`n8n-nodes-base.code`). Filters out videos present in the seen registry, injects manual test fixtures if specified, and enforces run candidate caps.
  - **Configuration:** Compares incoming items against a `Set` of known `videoId` values from the sheet, applies `FORCE_CHANGE_FOR` override logic, and slices results to `maxCandidatesPerRun`.
  - **Expressions/Variables:** Reads configuration parameters and upstream sheet data.
  - **Connections:** Input from `Combine Parsed Video Data`; output connects to `Check for New Videos`.

- **Check for New Videos**
  - **Type & Role:** If node (`n8n-nodes-base.if`). Evaluates whether the candidate queue contains items for processing.
  - **Configuration:** Condition checks if `hasChanges` equals `true`.
  - **Expressions/Variables:** Evaluates `={{ $json.hasChanges }}`.
  - **Connections:** Input from `Deduplicate and Queue Candidates`; True branch routes to `Split Candidates for Processing`, False branch routes to `No New Videos Discovered`.

- **No New Videos Discovered**
  - **Type & Role:** No-Op node (`n8n-nodes-base.noOp`). Acts as a terminal endpoint for empty runs.
  - **Connections:** Input from `Check for New Videos` (False branch).

---

#### 2.5 AI Generation & Processing
This block iterates through approved video candidates, interacts with Google Gemini via LangChain to generate structured content ideas, and parses the AI response.

- **Split Candidates for Processing**
  - **Type & Role:** Code node (`n8n-nodes-base.code`). Unpacks the candidate array into distinct individual items for looping.
  - **Connections:** Input from `Check for New Videos` (True branch); output connects to `Batch Process Candidates`.

- **Batch Process Candidates**
  - **Type & Role:** Split In Batches node (`n8n-nodes-base.splitInBatches`). Iterates over each candidate video sequentially.
  - **Connections:** Input from `Split Candidates for Processing` and `Process AI Output Ideas`; loop output branches to `AI Generate Content Ideas` (looping) and `Compile Run Summary` (completion).

- **AI Generate Content Ideas**
  - **Type & Role:** Advanced AI Chain LLM node (`@n8n/n8n-nodes-langchain.chainLlm`). Sends competitor video metadata to the language model with strict JSON formatting instructions.
  - **Configuration:** Prompt instructs the model to act as a YouTube content strategist returning minified JSON containing `ideaTitles`, `opportunityScore`, `differentiation`, `contentFormat`, and `reasoning`.
  - **Expressions/Variables:** Injects `{{ $json.channelId }}`, `{{ $json.title }}`, and `{{ $json.publishedAt }}`.
  - **Connections:** Connected to `Use Gemini Chat Model` via AI language model input; input from `Batch Process Candidates`; output connects to `Process AI Output Ideas`.
  - **Edge Cases/Failures:** LLM service degradation, token limits, or refusal errors.

- **Use Gemini Chat Model**
  - **Type & Role:** Google Gemini Chat Model node (`@n8n/n8n-nodes-langchain.lmChatGoogleGemini`). Provides the underlying model configuration for the LLM chain.
  - **Configuration:** Model set to `models/gemini-3.1-flash-lite`.
  - **Credentials:** Google Gemini (PaLM) API account.
  - **Connections:** Output connects to `AI Generate Content Ideas`.

- **Process AI Output Ideas**
  - **Type & Role:** Code node (`n8n-nodes-base.code`). Normalizes and parses raw text responses from the LLM into a structured schema, providing fallback defaults if parsing fails.
  - **Configuration:** JavaScript parsing routine stripping markdown code fences and matching JSON objects, falling back to an opportunity score of 5 if invalid.
  - **Connections:** Input from `AI Generate Content Ideas`; output fans out to `Evaluate Idea Score`, `Create Seen Video Entry`, and `Batch Process Candidates` (to continue loop).
  - **Edge Cases/Failures:** Malformed LLM output lacking valid JSON structures (handled by fallback catch block).

---

#### 2.6 Evaluation, Logging & Persistence
This block filters ideas based on quality scores, records high-value concepts to the ideas sheet, and logs all processed videos to the seen registry.

- **Evaluate Idea Score**
  - **Type & Role:** If node (`n8n-nodes-base.if`). Checks whether an idea's opportunity score meets or exceeds the threshold.
  - **Configuration:** Numeric comparison verifying `opportunityScore >= ideaScoreThreshold`.
  - **Expressions/Variables:** Evaluates `={{ $json.opportunityScore }}` against `={{ $('Configure Scan Parameters').first().json.ideaScoreThreshold }}`.
  - **Connections:** Input from `Process AI Output Ideas`; True branch connects to `Append Idea to Spreadsheet`, False branch connects to `Discard Low-Score Ideas`.

- **Discard Low-Score Ideas**
  - **Type & Role:** No-Op node (`n8n-nodes-base.noOp`). Terminal endpoint for rejected ideas.
  - **Connections:** Input from `Evaluate Idea Score` (False branch).

- **Append Idea to Spreadsheet**
  - **Type & Role:** Google Sheets node (`n8n-nodes-base.googleSheets`). Appends qualifying content ideas to the designated tracking sheet.
  - **Configuration:** Operation set to `append` with auto-mapped input columns.
  - **Expressions/Variables:** `documentId` and `sheetName` reference configuration parameters.
  - **Connections:** Input from `Evaluate Idea Score` (True branch).
  - **Credentials:** Google Sheets OAuth2 API.
  - **Edge Cases/Failures:** Spreadsheet API write limits or schema mismatches.

- **Create Seen Video Entry**
  - **Type & Role:** Code node (`n8n-nodes-base.code`). Shapes a concise record for tracking processed videos.
  - **Configuration:** Extracts `videoId`, `channelId`, `title`, and appends an ISO timestamp `seenAt`.
  - **Connections:** Input from `Process AI Output Ideas`; output connects to `Log Video in Sheets`.

- **Log Video in Sheets**
  - **Type & Role:** Google Sheets node (`n8n-nodes-base.googleSheets`). Appends the processed video record to the "Seen Videos" tracking tab.
  - **Configuration:** Operation set to `append` with auto-mapped input columns.
  - **Expressions/Variables:** References configuration parameters for document and sheet names.
  - **Connections:** Input from `Create Seen Video Entry`.
  - **Credentials:** Google Sheets OAuth2 API.
  - **Edge Cases/Failures:** Network timeouts or API write restrictions.

---

#### 2.7 Digest Compilation & Notification
This block aggregates run results, determines if an email notification is required, and sends the summary digest.

- **Compile Run Summary**
  - **Type & Role:** Code node (`n8n-nodes-base.code`). Gathers all saved ideas from the spreadsheet append step to compile a consolidated summary.
  - **Configuration:** Reads execution history from `Append Idea to Spreadsheet` using `.all()`.
  - **Connections:** Input from `Batch Process Candidates` (completion branch); output connects to `Check Ideas for Reporting`.

- **Check Ideas for Reporting**
  - **Type & Role:** If node (`n8n-nodes-base.if`). Evaluates whether any ideas were successfully saved during the execution run.
  - **Configuration:** Condition checks if `hasIdeas` equals `true`.
  - **Expressions/Variables:** Evaluates `={{ $json.hasIdeas }}`.
  - **Connections:** Input from `Compile Run Summary`; True branch routes to `Construct Email Summary`, False branch routes to `No Ideas Ready to Report`.

- **Construct Email Summary**
  - **Type & Role:** Code node (`n8n-nodes-base.code`). Formats the text content and subject line for the email notification digest.
  - **Configuration:** JavaScript mapping utility building numbered lists containing idea titles, formats, scores, reasoning, and source links.
  - **Connections:** Input from `Check Ideas for Reporting` (True branch); output connects to `Email Content Idea Digest`.

- **Email Content Idea Digest**
  - **Type & Role:** Gmail node (`n8n-nodes-base.gmail`). Sends the plain-text summary email to the designated recipient.
  - **Configuration:** Sends plain text messages using dynamic subject and body fields.
  - **Expressions/Variables:** `sendTo` uses `={{ $('Configure Scan Parameters').first().json.notifyEmail }}`, subject and message reference upstream code node outputs.
  - **Connections:** Input from `Construct Email Summary`.
  - **Credentials:** Gmail OAuth2 account.
  - **Edge Cases/Failures:** Invalid recipient addresses or revoked OAuth credentials.

- **No Ideas Ready to Report**
  - **Type & Role:** No-Op node (`n8n-nodes-base.noOp`). Terminal endpoint when no reportable ideas are generated.
  - **Connections:** Input from `Check Ideas for Reporting` (False branch).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation overview | None | None | ## YouTube Competitor Idea Miner<br><br>### How it works<br><br>This workflow scans competitor YouTube channels on a weekly schedule or from a manual form submission, reads each channel’s RSS feed, and filters out videos that have already been processed. For each new candidate video, it uses Gemini to generate content ideas, scores them, saves high-scoring ideas to Google Sheets, and records every processed video as seen. After the candidate loop finishes, it builds a run summary and emails an idea digest when there are reportable ideas.<br><br>### Setup steps<br><br>- Configure the schedule trigger and/or form trigger inputs for how scans should be started.<br>- Update the scan configuration with competitorChannelIds, maxVideosPerChannel, maxCandidatesPerRun, and ideaScoreThreshold.<br>- Connect Google Sheets credentials and configure the sheets/ranges used for seen video IDs, saved ideas, and processed video records.<br>- Connect Google Gemini credentials for the AI idea-generation chain and verify the prompt/model settings.<br>- Connect Gmail credentials and set the digest email recipient, sender, and subject details.<br><br>### Customization<br><br>Adjust the competitor channel list, scoring threshold, candidate limits, AI prompt, and email digest format to match the niche and publishing strategy. |
| Sticky Note1 | n8n-nodes-base.stickyNote | Scan configuration grouping | None | None | ## Start scan configuration<br><br>Starts the workflow from either the weekly schedule or form submission, then sets the scan parameters used by the rest of the run. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Seen videos grouping | None | None | ## Load seen videos<br><br>Reads the existing seen-video registry from Google Sheets so later steps can avoid reprocessing the same competitor uploads. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Channel video fetching grouping | None | None | ## Fetch channel videos<br><br>Splits the configured competitor channel list, loops through each channel, fetches its YouTube RSS feed, and parses the feed into video records. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Candidate queue grouping | None | None | ## Build candidate queue<br><br>Combines all parsed channel videos, removes duplicates and already-seen items, then checks whether there are any new videos to process. |
| Sticky Note5 | n8n-nodes-base.stickyNote | No new videos path grouping | None | None | ## No new videos path<br><br>Ends the run cleanly when the scan finds no new competitor videos. |
| Sticky Note6 | n8n-nodes-base.stickyNote | AI idea generation grouping | None | None | ## Generate AI ideas<br><br>Splits the candidate queue, loops over each candidate video, sends it to Gemini for idea generation, and parses the AI response before continuing the loop. |
| Sticky Note7 | n8n-nodes-base.stickyNote | Filter and save ideas grouping | None | None | ## Filter and save ideas<br><br>Checks whether each generated idea meets the score threshold, saves qualifying ideas to Google Sheets, and skips low-scoring ideas. |
| Sticky Note8 | n8n-nodes-base.stickyNote | Mark videos seen grouping | None | None | ## Mark videos seen<br><br>Builds a seen-video record for each processed candidate and writes it to Google Sheets so future scans can ignore it. |
| Sticky Note9 | n8n-nodes-base.stickyNote | Run digest grouping | None | None | ## Prepare run digest<br><br>After all candidates are processed, builds the run summary, decides whether there are ideas worth reporting, prepares the email body, or exits if nothing should be sent. |
| Sticky Note10 | n8n-nodes-base.stickyNote | Send digest email grouping | None | None | ## Send digest email<br><br>Sends the prepared idea digest through Gmail to notify the user about the best new content opportunities. |
| Weekly Channel Scan Trigger | n8n-nodes-base.scheduleTrigger | Scheduled trigger | None | Configure Scan Parameters | ## Start scan configuration<br><br>Starts the workflow from either the weekly schedule or form submission, then sets the scan parameters used by the rest of the run. |
| Configure Scan Parameters | n8n-nodes-base.set | Parameter initialization | Weekly Channel Scan Trigger, Form Submission Trigger | Read Seen Video IDs from Sheet | ## Start scan configuration<br><br>Starts the workflow from either the weekly schedule or form submission, then sets the scan parameters used by the rest of the run. |
| Read Seen Video IDs from Sheet | n8n-nodes-base.googleSheets | Read historical state | Configure Scan Parameters | Split Channel ID List | ## Load seen videos<br><br>Reads the existing seen-video registry from Google Sheets so later steps can avoid reprocessing the same competitor uploads. |
| Split Channel ID List | n8n-nodes-base.code | Channel parsing | Read Seen Video IDs from Sheet | Batch Process Channels | ## Fetch channel videos<br><br>Splits the configured competitor channel list, loops through each channel, fetches its YouTube RSS feed, and parses the feed into video records. |
| Batch Process Channels | n8n-nodes-base.splitInBatches | Channel iteration controller | Split Channel ID List, Parse Videos from RSS | Combine Parsed Video Data, Fetch YouTube Channel RSS | ## Fetch channel videos<br><br>Splits the configured competitor channel list, loops through each channel, fetches its YouTube RSS feed, and parses the feed into video records. |
| Fetch YouTube Channel RSS | n8n-nodes-base.httpRequest | HTTP feed retrieval | Batch Process Channels | Parse Videos from RSS | ## Fetch channel videos<br><br>Splits the configured competitor channel list, loops through each channel, fetches its YouTube RSS feed, and parses the feed into video records. |
| Parse Videos from RSS | n8n-nodes-base.code | RSS XML parser | Fetch YouTube Channel RSS | Batch Process Channels | ## Fetch channel videos<br><br>Splits the configured competitor channel list, loops through each channel, fetches its YouTube RSS feed, and parses the feed into video records. |
| Combine Parsed Video Data | n8n-nodes-base.code | Aggregate channel items | Batch Process Channels | Deduplicate and Queue Candidates | ## Build candidate queue<br><br>Combines all parsed channel videos, removes duplicates and already-seen items, then checks whether there are any new videos to process. |
| Deduplicate and Queue Candidates | n8n-nodes-base.code | Filtering and capping | Combine Parsed Video Data | Check for New Videos | ## Build candidate queue<br><br>Combines all parsed channel videos, removes duplicates and already-seen items, then checks whether there are any new videos to process. |
| Check for New Videos | n8n-nodes-base.if | Change detection branch | Deduplicate and Queue Candidates | Split Candidates for Processing, No New Videos Discovered | ## Build candidate queue<br><br>Combines all parsed channel videos, removes duplicates and already-seen items, then checks whether there are any new videos to process. |
| No New Videos Discovered | n8n-nodes-base.noOp | Terminal empty path | Check for New Videos | None | ## No new videos path<br><br>Ends the run cleanly when the scan finds no new competitor videos. |
| Split Candidates for Processing | n8n-nodes-base.code | Candidate unpacking | Check for New Videos | Batch Process Candidates | ## Generate AI ideas<br><br>Splits the candidate queue, loops over each candidate video, sends it to Gemini for idea generation, and parses the AI response before continuing the loop. |
| Batch Process Candidates | n8n-nodes-base.splitInBatches | Candidate iteration controller | Split Candidates for Processing, Process AI Output Ideas | Compile Run Summary, AI Generate Content Ideas | ## Generate AI ideas<br><br>Splits the candidate queue, loops over each candidate video, sends it to Gemini for idea generation, and parses the AI response before continuing the loop. |
| AI Generate Content Ideas | @n8n/n8n-nodes-langchain.chainLlm | AI generation chain | Batch Process Candidates, Use Gemini Chat Model | Process AI Output Ideas | ## Generate AI ideas<br><br>Splits the candidate queue, loops over each candidate video, sends it to Gemini for idea generation, and parses the AI response before continuing the loop. |
| Use Gemini Chat Model | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | LLM model provider | None | AI Generate Content Ideas | ## Generate AI ideas<br><br>Splits the candidate queue, loops over each candidate video, sends it to Gemini for idea generation, and parses the AI response before continuing the loop. |
| Process AI Output Ideas | n8n-nodes-base.code | AI output normalization | AI Generate Content Ideas | Evaluate Idea Score, Create Seen Video Entry, Batch Process Candidates | ## Generate AI ideas<br><br>Splits the candidate queue, loops over each candidate video, sends it to Gemini for idea generation, and parses the AI response before continuing the loop. |
| Evaluate Idea Score | n8n-nodes-base.if | Threshold verification | Process AI Output Ideas | Append Idea to Spreadsheet, Discard Low-Score Ideas | ## Filter and save ideas<br><br>Checks whether each generated idea meets the score threshold, saves qualifying ideas to Google Sheets, and skips low-scoring ideas. |
| Discard Low-Score Ideas | n8n-nodes-base.noOp | Terminal rejection path | Evaluate Idea Score | None | ## Filter and save ideas<br><br>Checks whether each generated idea meets the score threshold, saves qualifying ideas to Google Sheets, and skips low-scoring ideas. |
| Append Idea to Spreadsheet | n8n-nodes-base.googleSheets | Idea storage write | Evaluate Idea Score | None | ## Filter and save ideas<br><br>Checks whether each generated idea meets the score threshold, saves qualifying ideas to Google Sheets, and skips low-scoring ideas. |
| Create Seen Video Entry | n8n-nodes-base.code | Seen record builder | Process AI Output Ideas | Log Video in Sheets | ## Mark videos seen<br><br>Builds a seen-video record for each processed candidate and writes it to Google Sheets so future scans can ignore it. |
| Log Video in Sheets | n8n-nodes-base.googleSheets | Seen storage write | Create Seen Video Entry | None | ## Mark videos seen<br><br>Builds a seen-video record for each processed candidate and writes it to Google Sheets so future scans can ignore it. |
| Compile Run Summary | n8n-nodes-base.code | Summary aggregation | Batch Process Candidates | Check Ideas for Reporting | ## Prepare run digest<br><br>After all candidates are processed, builds the run summary, decides whether there are ideas worth reporting, prepares the email body, or exits if nothing should be sent. |
| Check Ideas for Reporting | n8n-nodes-base.if | Notification evaluation | Compile Run Summary | Construct Email Summary, No Ideas Ready to Report | ## Prepare run digest<br><br>After all candidates are processed, builds the run summary, decides whether there are ideas worth reporting, prepares the email body, or exits if nothing should be sent. |
| Construct Email Summary | n8n-nodes-base.code | Email formatter | Check Ideas for Reporting | Email Content Idea Digest | ## Prepare run digest<br><br>After all candidates are processed, builds the run summary, decides whether there are ideas worth reporting, prepares the email body, or exits if nothing should be sent. |
| Email Content Idea Digest | n8n-nodes-base.gmail | Email dispatcher | Construct Email Summary | None | ## Send digest email<br><br>Sends the prepared idea digest through Gmail to notify the user about the best new content opportunities. |
| No Ideas Ready to Report | n8n-nodes-base.noOp | Terminal quiet path | Check Ideas for Reporting | None | ## Prepare run digest<br><br>After all candidates are processed, builds the run summary, decides whether there are ideas worth reporting, prepares the email body, or exits if nothing should be sent. |
| Form Submission Trigger | n8n-nodes-base.formTrigger | Manual web trigger | None | Configure Scan Parameters | ## Start scan configuration<br><br>Starts the workflow from either the weekly schedule or form submission, then sets the scan parameters used by the rest of the run. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Triggers & Configuration:**
   - Add a **Schedule Trigger** (`Weekly Channel Scan Trigger`) set to execute weekly on Mondays at 08:00 AM.
   - Add a **Form Trigger** (`Form Submission Trigger`) with a required text field named `Channel IDs`.
   - Add a **Set** node (`Configure Scan Parameters`). Connect both triggers to this node. Configure assignments: `competitorChannelIds` = `={{ $json['Channel IDs'] }}` (or hardcoded string), `maxVideosPerChannel` = `5`, `maxCandidatesPerRun` = `10`, `ideaScoreThreshold` = `7`, `spreadsheetId` = your spreadsheet ID, `seenVideosSheetName` = `Seen Videos`, `ideasSheetName` = `Video Ideas`, `notifyEmail` = your email, `FORCE_CHANGE_FOR` = `10 Productivity Hacks Nobody Talks About`.

2. **Set Up Seen State Reader:**
   - Add a **Google Sheets** node (`Read Seen Video IDs from Sheet`). Connect `Configure Scan Parameters` to it. Set operation to read, document ID and sheet name referencing the configuration variables. Enable `Continue Regular Output on Error`.
   - Connect the output to a **Code** node (`Split Channel ID List`) using JavaScript to split `competitorChannelIds` by commas into individual items.

3. **Build RSS Ingestion Loop:**
   - Add a **Split In Batches** node (`Batch Process Channels`). Connect `Split Channel ID List` to its input.
   - Add an **HTTP Request** node (`Fetch YouTube Channel RSS`). Set method to GET, URL to `=https://www.youtube.com/feeds/videos.xml?channel_id={{ $json.channelId }}`, response format to text, and enable `Never Error`. Connect loop output 1 of `Batch Process Channels` to this node.
   - Add a **Code** node (`Parse Videos from RSS`) to extract XML tags (`<yt:videoId>`, `<title>`, `<published>`, `<link>`). Connect `Fetch YouTube Channel RSS` to it. Connect its output back to `Batch Process Channels` to complete the loop.
   - Connect loop output 2 of `Batch Process Channels` to a **Code** node (`Combine Parsed Video Data`) using `.all()` to gather all parsed items.

4. **Construct Candidate Queue & Filtering:**
   - Connect `Combine Parsed Video Data` to a **Code** node (`Deduplicate and Queue Candidates`) to filter out IDs present in the seen sheet, apply the test override, and cap candidates.
   - Connect the output to an **If** node (`Check for New Videos`) evaluating `={{ $json.hasChanges }}` equals `true`.
   - Connect the False branch to a **No-Op** node (`No New Videos Discovered`).

5. **Set Up AI Generation Loop:**
   - Connect the True branch of `Check for New Videos` to a **Code** node (`Split Candidates for Processing`) to unpack candidate arrays.
   - Add a **Split In Batches** node (`Batch Process Candidates`). Connect `Split Candidates for Processing` and `Process AI Output Ideas` to its input.
   - Add an **Advanced AI Chain LLM** node (`AI Generate Content Ideas`) connected to loop output 1 of `Batch Process Candidates`. Set prompt to request minified JSON with specified fields (`ideaTitles`, `opportunityScore`, `differentiation`, `contentFormat`, `reasoning`).
   - Add a **Google Gemini Chat Model** node (`Use Gemini Chat Model`) configured with model `models/gemini-3.1-flash-lite` and valid Google Gemini (PaLM) API credentials. Connect it to the `ai_languageModel` input of `AI Generate Content Ideas`.
   - Add a **Code** node (`Process AI Output Ideas`) to parse and sanitize the LLM JSON response. Connect `AI Generate Content Ideas` to it. Connect its output back to `Batch Process Candidates` to continue the loop.

6. **Implement Evaluation, Persistence & Notification:**
   - Connect the output of `Process AI Output Ideas` to:
     - An **If** node (`Evaluate Idea Score`) evaluating `={{ $json.opportunityScore }} >= {{ $('Configure Scan Parameters').first().json.ideaScoreThreshold }}`.
       - True branch connects to a **Google Sheets** node (`Append Idea to Spreadsheet`) set to append to the ideas sheet.
       - False branch connects to a **No-Op** node (`Discard Low-Score Ideas`).
     - A **Code** node (`Create Seen Video Entry`) shaping seen records, which connects to a **Google Sheets** node (`Log Video in Sheets`) set to append to the seen videos sheet.
   - Connect loop output 2 of `Batch Process Candidates` to a **Code** node (`Compile Run Summary`) reading from `Append Idea to Spreadsheet`.
   - Connect `Compile Run Summary` to an **If** node (`Check Ideas for Reporting`) evaluating `={{ $json.hasIdeas }}` equals `true`.
     - True branch connects to a **Code** node (`Construct Email Summary`), which connects to a **Gmail** node (`Email Content Idea Digest`) sending the message to `notifyEmail` using Gmail OAuth2 credentials.
     - False branch connects to a **No-Op** node (`No Ideas Ready to Report`).

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| This workflow scans competitor YouTube channels via RSS feeds, processes them with Google Gemini, logs data to Google Sheets, and sends email digests via Gmail. | General Workflow Summary |
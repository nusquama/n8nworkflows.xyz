Send personalized daily learning emails for women using GPT-4o and Gmail

https://n8nworkflows.xyz/workflows/send-personalized-daily-learning-emails-for-women-using-gpt-4o-and-gmail-14021


# Send personalized daily learning emails for women using GPT-4o and Gmail

# 1. Workflow Overview

This workflow sends a personalized daily learning email to each subscriber in a women-focused learning program. It combines Google Sheets as the subscriber/content database, GPT-4o for individualized lesson generation, Gmail for delivery, and a logging sheet for traceability.

Its main use case is daily educational email delivery at scale where each subscriber receives a different message based on:

- their topic interest
- how long they have been subscribed
- their inferred learning phase: Beginner, Intermediate, or Advanced

The workflow is organized into the following logical blocks.

## 1.1 Scheduled Data Intake

At 9:00 AM every day, the workflow starts automatically, loads all subscribers from the `Subscribers` sheet, and computes timing-related fields such as day number, week number, subscribed days, and learning phase.

## 1.2 Content Retrieval and Lesson Matching

The workflow loads the full `ContentLibrary` sheet once, then scores each lesson against each subscriber’s interest and phase to pick the most relevant lesson. If no suitable match is found, it falls back to a rotating lesson based on day-of-year index.

## 1.3 Per-Subscriber AI Personalization

Matched subscriber/lesson records are processed one at a time through a batch loop. For each subscriber, GPT-4o receives the learner profile and selected lesson content and is asked to return strict JSON containing a personalized explanation, task, quote, and takeaways.

## 1.4 Email Assembly, Delivery, and Logging

The AI output is parsed and merged into a branded HTML email. The email is sent through Gmail, then the delivery is appended to a `SendLog` sheet for tracking.

---

# 2. Block-by-Block Analysis

## 2.1 Scheduled Data Intake

### Overview
This block is the entry point of the workflow. It runs every day at 9:00 AM, loads all subscribers, and enriches each subscriber with calculated timing and progression fields used later for lesson matching and email personalization.

### Nodes Involved
- Daily 9AM Trigger
- Fetch All Subscribers
- Calculate Day, Week & Phase

### Node Details

#### Daily 9AM Trigger
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`; workflow entry node using a cron schedule.
- **Configuration choices:** Uses cron expression `0 9 * * *`, which means every day at 09:00.
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input: none  
  - Output: `Fetch All Subscribers`
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases or potential failure types:**
  - Timezone behavior depends on the n8n instance timezone.
  - If the server is down at the scheduled moment, execution may be missed depending on deployment behavior.
- **Sub-workflow reference:** None.

#### Fetch All Subscribers
- **Type and technical role:** `n8n-nodes-base.googleSheets`; reads rows from the `Subscribers` worksheet.
- **Configuration choices:**
  - Uses Google Sheets OAuth2 credentials named `automations`
  - Spreadsheet: `Women Skill Learning Automation`
  - Sheet: `Subscribers` (`gid=0`)
  - Default read operation behavior is used
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input: `Daily 9AM Trigger`  
  - Output: `Calculate Day, Week & Phase`
- **Version-specific requirements:** Type version `4.4`.
- **Edge cases or potential failure types:**
  - OAuth token expiration or insufficient scopes
  - Wrong spreadsheet or sheet selection
  - Missing expected columns such as `Name`, `Email`, `Topic Interest`, `Subscribed Date`
  - Empty sheet returns no subscriber items, causing no downstream personalization
- **Sub-workflow reference:** None.

#### Calculate Day, Week & Phase
- **Type and technical role:** `n8n-nodes-base.code`; computes time-derived fields for each subscriber.
- **Configuration choices:**
  - JavaScript code calculates:
    - `dayOfYear`
    - `weekNumber`
    - `subscribedDays`
    - `phase`
  - Phase logic:
    - `Beginner` by default
    - `Intermediate` if subscribed more than 30 days
    - `Advanced` if subscribed more than 90 days
- **Key expressions or variables used:**
  - `item.json['Subscribed Date']`
  - `now`
  - `oneDay = 86400000`
- **Input and output connections:**  
  - Input: `Fetch All Subscribers`  
  - Output: `Fetch Content Library`
- **Version-specific requirements:** Code node type version `2`.
- **Edge cases or potential failure types:**
  - Invalid or empty `Subscribed Date` falls back to current date, making `subscribedDays` equal to 0
  - Date parsing may vary if sheet values are not in ISO or locale-safe format
  - Week number is based on day-of-year / 7, not ISO week logic
  - `dayOfYear` is global for current date, not subscriber-relative
- **Sub-workflow reference:** None.

---

## 2.2 Content Retrieval and Lesson Matching

### Overview
This block loads the full content catalog and evaluates each lesson against each subscriber. It chooses the highest-scoring lesson using topic and phase alignment, with a fallback mechanism based on the day-of-year index.

### Nodes Involved
- Fetch Content Library
- Match Subscriber to Lesson

### Node Details

#### Fetch Content Library
- **Type and technical role:** `n8n-nodes-base.googleSheets`; loads all content rows from the content library.
- **Configuration choices:**
  - Uses the same spreadsheet as subscribers
  - Sheet: `ContentLibrary`
  - `executeOnce` is enabled, so the sheet is fetched a single time for the run rather than per incoming item
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input: `Calculate Day, Week & Phase`  
  - Output: `Match Subscriber to Lesson`
- **Version-specific requirements:** Type version `4.4`.
- **Edge cases or potential failure types:**
  - Empty content library leaves no good lesson candidates
  - Missing expected columns such as `Category`, `Phase`, `Lesson Title`, `Raw Content`, `Resource Link`
  - Auth and permission issues with Google Sheets
- **Sub-workflow reference:** None.

#### Match Subscriber to Lesson
- **Type and technical role:** `n8n-nodes-base.code`; cross-matches all subscribers to all content items and emits one enriched item per subscriber.
- **Configuration choices:**
  - Pulls subscriber data directly using `$('Calculate Day, Week & Phase').all()`
  - Uses current input as the content library
  - Scoring logic:
    - +3 if category and topic interest contain each other
    - +2 if lesson phase exactly matches subscriber phase
  - Tracks `bestLesson` by highest score
  - Fallback: if no lesson is selected and content exists, choose `contentItems[dayOfYear % contentItems.length]`
- **Key expressions or variables used:**
  - `subscriber.json['Topic Interest']`
  - `subscriber.json['phase']`
  - `subscriber.json['dayOfYear']`
  - `content.json['Category']`
  - `content.json['Phase']`
- **Input and output connections:**  
  - Input: `Fetch Content Library`  
  - Output: `Loop Each Subscriber`
- **Version-specific requirements:** Code node type version `2`.
- **Edge cases or potential failure types:**
  - If categories and interests do not overlap textually, scoring may still pick a lesson with score `0`, because `bestScore` starts at `-1`
  - Fallback is rarely reached because any first content item with score `0` becomes the best lesson
  - Text matching is simplistic and substring-based, not semantic
  - If content sheet is empty, defaults are used in output fields
  - If subscriber fields are missing, output may contain empty `name`, `email`, or `topicInterest`
- **Sub-workflow reference:** None.

---

## 2.3 Per-Subscriber AI Personalization

### Overview
This block processes one matched subscriber at a time and sends a structured prompt to GPT-4o. The goal is to generate consistent JSON that can be parsed into a personalized lesson email.

### Nodes Involved
- Loop Each Subscriber
- Personalize Lesson

### Node Details

#### Loop Each Subscriber
- **Type and technical role:** `n8n-nodes-base.splitInBatches`; iterates through matched subscriber records one by one.
- **Configuration choices:**
  - Default batch behavior with no special options configured
  - In this workflow design, the loop is closed by routing from `Log Delivery to Sheets` back to this node
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input: `Match Subscriber to Lesson`
  - Output 0: unused in this workflow
  - Output 1: `Personalize Lesson`
  - Loop return input from: `Log Delivery to Sheets`
- **Version-specific requirements:** Type version `3`.
- **Edge cases or potential failure types:**
  - Split-in-batches routing can be confusing when rebuilding manually
  - If no items arrive, the AI/email branch is skipped
  - If a downstream node fails, later subscribers will not be processed unless error handling is added
- **Sub-workflow reference:** None.

#### Personalize Lesson
- **Type and technical role:** `@n8n/n8n-nodes-langchain.openAi`; calls OpenAI GPT-4o to generate a personalized lesson payload.
- **Configuration choices:**
  - Model: `gpt-4o`
  - Uses OpenAI credentials `OpenAi account 2`
  - Prompt includes:
    - learner name
    - topic interest
    - learning phase
    - day number
    - lesson title
    - raw content
  - System instruction requires:
    - women’s career coach voice
    - concise and actionable output
    - strict JSON only
- **Key expressions or variables used:**
  - `{{ $json.name }}`
  - `{{ $json.topicInterest }}`
  - `{{ $json.phase }}`
  - `{{ $json.dayOfYear }}`
  - `{{ $json.lessonTitle }}`
  - `{{ $json.lessonRawContent }}`
- **Input and output connections:**  
  - Input: `Loop Each Subscriber`
  - Output: `Build Personalized Email`
- **Version-specific requirements:** Type version `2.1`.
- **Edge cases or potential failure types:**
  - OpenAI credential or quota failure
  - Model availability issues
  - Non-JSON or malformed JSON response despite prompt constraints
  - Prompt injection risk if `Raw Content` contains adversarial instructions
  - Token size issues if lesson content is long
- **Sub-workflow reference:** None.

---

## 2.4 Email Assembly, Delivery, and Logging

### Overview
This block parses the AI response, builds a branded HTML email, sends the email with Gmail, and appends a delivery record to Google Sheets. It also closes the batch loop to continue with the next subscriber.

### Nodes Involved
- Build Personalized Email
- Send Daily Lesson Email
- Log Delivery to Sheets

### Node Details

#### Build Personalized Email
- **Type and technical role:** `n8n-nodes-base.code`; parses AI JSON and merges it with subscriber/lesson metadata into final HTML email content.
- **Configuration choices:**
  - Attempts to extract AI text from `item.json.output?.[0]?.content?.[0]?.text`
  - Removes fenced code markers like ```json
  - Uses `JSON.parse`
  - If parsing fails, falls back to default motivational content
  - Builds a full HTML email with inline CSS
  - Pulls subscriber data using `$('Match Subscriber to Lesson').item.json`
- **Key expressions or variables used:**
  - `item.json.output?.[0]?.content?.[0]?.text`
  - `$('Match Subscriber to Lesson').item.json`
  - `resourceLink`, `lessonCategory`, `phase`, `dayOfYear`, etc.
- **Input and output connections:**  
  - Input: `Personalize Lesson`
  - Output: `Send Daily Lesson Email`
- **Version-specific requirements:** Code node type version `2`.
- **Edge cases or potential failure types:**
  - The expression `$('Match Subscriber to Lesson').item.json` may not reliably resolve the correct paired item in some execution contexts; paired item linking should be verified
  - HTML is assembled without escaping user or AI content, so broken markup is possible if generated text contains problematic HTML
  - Fallback quote is attributed to Ayn Rand, which may not align with the “woman leader in their field” requirement
  - If AI returns `keyTakeaways` as non-array, list rendering may break or be empty
- **Sub-workflow reference:** None.

#### Send Daily Lesson Email
- **Type and technical role:** `n8n-nodes-base.gmail`; sends an email via Gmail OAuth2.
- **Configuration choices:**
  - Recipient: `={{ $json.email }}`
  - Message body: `={{ $json.htmlEmail }}`
  - Subject intended to include day number, category, and lesson title
- **Key expressions or variables used:**
  - `{{ $json.email }}`
  - `{{ $json.htmlEmail }}`
  - Subject: `🌟 Day {{$json.dayOfYear }} | {{ $json.lessonCategory }} : {{ $json.lessonTitle }}}}`
- **Input and output connections:**  
  - Input: `Build Personalized Email`
  - Output: `Log Delivery to Sheets`
- **Version-specific requirements:** Type version `2.1`.
- **Edge cases or potential failure types:**
  - **Important configuration issue:** the subject expression has extra closing braces `}}}}`, which can cause expression parsing failure or malformed subject output
  - Gmail OAuth permissions or token expiration
  - Gmail sending quotas / rate limits
  - Depending on node configuration defaults, the HTML body may need explicit HTML formatting support verification
  - Invalid subscriber email addresses will fail delivery
- **Sub-workflow reference:** None.

#### Log Delivery to Sheets
- **Type and technical role:** `n8n-nodes-base.googleSheets`; appends one row per delivered email to `SendLog`.
- **Configuration choices:**
  - Operation: `append`
  - Target sheet: `SendLog`
  - Columns mapped manually:
    - `Date`
    - `Phase`
    - `Category`
    - `AI Snippet`
    - `Day Number`
    - `Lesson Title`
    - `Subscriber Name`
    - `Subscriber Email`
- **Key expressions or variables used:**
  - `{{ $now.toISO() }}`
  - `{{ $('Build Personalized Email').item.json.phase }}`
  - similar expressions for lesson/category/snippet/day/title
- **Input and output connections:**  
  - Input: `Send Daily Lesson Email`
  - Output: `Loop Each Subscriber` to continue batch iteration
- **Version-specific requirements:** Type version `4.4`.
- **Edge cases or potential failure types:**
  - **Important mapping issue:** `Subscriber Name` is mapped from email and `Subscriber Email` is mapped from name; these two fields are reversed
  - Google Sheets auth or permission errors
  - Schema mismatch if target columns do not exist
  - Logging occurs only after successful Gmail node output; failed sends will not be logged unless alternate error logging is added
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Daily 9AM Trigger | Schedule Trigger | Starts workflow daily at 9 AM |  | Fetch All Subscribers | Runs daily at 9AM. Fetches all subscribers and the full content library. Calculates each subscriber's day number, week number, and learning phase based on how long they've been subscribed. |
| Fetch All Subscribers | Google Sheets | Reads subscriber records from the Subscribers sheet | Daily 9AM Trigger | Calculate Day, Week & Phase | Runs daily at 9AM. Fetches all subscribers and the full content library. Calculates each subscriber's day number, week number, and learning phase based on how long they've been subscribed. |
| Calculate Day, Week & Phase | Code | Computes day/year metrics and assigns learning phase | Fetch All Subscribers | Fetch Content Library | Runs daily at 9AM. Fetches all subscribers and the full content library. Calculates each subscriber's day number, week number, and learning phase based on how long they've been subscribed. |
| Fetch Content Library | Google Sheets | Reads lesson catalog from the ContentLibrary sheet | Calculate Day, Week & Phase | Match Subscriber to Lesson | Runs daily at 9AM. Fetches all subscribers and the full content library. Calculates each subscriber's day number, week number, and learning phase based on how long they've been subscribed. |
| Match Subscriber to Lesson | Code | Scores and selects the best lesson for each subscriber | Fetch Content Library | Loop Each Subscriber | Matches each subscriber to the best lesson based on topic interest and phase. Loops one by one and sends each profile to GPT-4o which generates a personalized lesson, task, takeaways, and a motivational quote from a woman leader. |
| Loop Each Subscriber | Split In Batches | Processes matched subscriber items one at a time | Match Subscriber to Lesson; Log Delivery to Sheets | Personalize Lesson | Matches each subscriber to the best lesson based on topic interest and phase. Loops one by one and sends each profile to GPT-4o which generates a personalized lesson, task, takeaways, and a motivational quote from a woman leader. |
| Personalize Lesson | OpenAI (LangChain) | Generates structured personalized lesson JSON with GPT-4o | Loop Each Subscriber | Build Personalized Email | Matches each subscriber to the best lesson based on topic interest and phase. Loops one by one and sends each profile to GPT-4o which generates a personalized lesson, task, takeaways, and a motivational quote from a woman leader. |
| Build Personalized Email | Code | Parses AI JSON and renders branded HTML email | Personalize Lesson | Send Daily Lesson Email | Builds a branded HTML email with all AI generated content and sends it to each subscriber via Gmail. Every delivery is logged to the SendLog sheet with date, name, lesson title, phase, and AI snippet. |
| Send Daily Lesson Email | Gmail | Sends the personalized HTML email to the subscriber | Build Personalized Email | Log Delivery to Sheets | Builds a branded HTML email with all AI generated content and sends it to each subscriber via Gmail. Every delivery is logged to the SendLog sheet with date, name, lesson title, phase, and AI snippet. |
| Log Delivery to Sheets | Google Sheets | Appends delivery metadata to the SendLog sheet | Send Daily Lesson Email | Loop Each Subscriber | Builds a branded HTML email with all AI generated content and sends it to each subscriber via Gmail. Every delivery is logged to the SendLog sheet with date, name, lesson title, phase, and AI snippet. |
| Sticky Note | Sticky Note | Documentation/comment node |  |  | ## Workflow Overview Every woman learns differently. This workflow sends a fully personalized daily lesson to every subscriber based on their topic interest and how long they've been learning — Beginner, Intermediate, or Advanced. No two emails are the same. |
| Sticky Note1 | Sticky Note | Documentation/comment node |  |  | Runs daily at 9AM. Fetches all subscribers and the full content library. Calculates each subscriber's day number, week number, and learning phase based on how long they've been subscribed. |
| Sticky Note2 | Sticky Note | Documentation/comment node |  |  | Matches each subscriber to the best lesson based on topic interest and phase. Loops one by one and sends each profile to GPT-4o which generates a personalized lesson, task, takeaways, and a motivational quote from a woman leader. |
| Sticky Note3 | Sticky Note | Documentation/comment node |  |  | Builds a branded HTML email with all AI generated content and sends it to each subscriber via Gmail. Every delivery is logged to the SendLog sheet with date, name, lesson title, phase, and AI snippet. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Automate Personalized Learning Emails for Women Using GPT-4o and Google Sheets`.

2. **Add a Schedule Trigger node**
   - Node type: `Schedule Trigger`
   - Name: `Daily 9AM Trigger`
   - Set schedule rule to cron
   - Enter cron expression: `0 9 * * *`
   - Confirm your n8n instance timezone is the intended timezone for daily sends.

3. **Prepare the Google Sheet**
   - Create one spreadsheet with three tabs:
     - `Subscribers`
     - `ContentLibrary`
     - `SendLog`
   - Suggested columns:
     - `Subscribers`: `Name`, `Email`, `Topic Interest`, `Subscribed Date`
     - `ContentLibrary`: `Lesson Title`, `Category`, `Phase`, `Raw Content`, `Resource Link`
     - `SendLog`: `Date`, `Subscriber Email`, `Subscriber Name`, `Lesson Title`, `Category`, `Phase`, `Day Number`, `AI Snippet`

4. **Add the first Google Sheets node**
   - Node type: `Google Sheets`
   - Name: `Fetch All Subscribers`
   - Connect `Daily 9AM Trigger -> Fetch All Subscribers`
   - Configure Google Sheets OAuth2 credentials
   - Select your spreadsheet
   - Select the `Subscribers` sheet
   - Use the default row-reading operation.

5. **Add a Code node for subscriber progression**
   - Node type: `Code`
   - Name: `Calculate Day, Week & Phase`
   - Connect `Fetch All Subscribers -> Calculate Day, Week & Phase`
   - Paste logic that:
     - computes current `dayOfYear`
     - computes `weekNumber`
     - calculates `subscribedDays` from `Subscribed Date`
     - assigns:
       - `Beginner` for 0–30 days
       - `Intermediate` for 31–90 days
       - `Advanced` for over 90 days
   - Output one item per subscriber with original fields plus computed fields.

6. **Add the second Google Sheets node**
   - Node type: `Google Sheets`
   - Name: `Fetch Content Library`
   - Connect `Calculate Day, Week & Phase -> Fetch Content Library`
   - Use the same Google Sheets OAuth2 credentials
   - Select the same spreadsheet
   - Select the `ContentLibrary` sheet
   - Enable `Execute Once` so the content library loads a single time for the whole run.

7. **Add a Code node for lesson matching**
   - Node type: `Code`
   - Name: `Match Subscriber to Lesson`
   - Connect `Fetch Content Library -> Match Subscriber to Lesson`
   - In the code:
     - get all subscribers from the earlier node using node reference
     - get all content items from current input
     - for each subscriber, score each lesson:
       - +3 for category/topic textual match
       - +2 for exact phase match
     - select the highest-scoring lesson
     - if needed, fallback to index rotation using `dayOfYear % contentItems.length`
   - Output fields should include:
     - `name`
     - `email`
     - `topicInterest`
     - `phase`
     - `dayOfYear`
     - `weekNumber`
     - `subscribedDays`
     - `lessonTitle`
     - `lessonCategory`
     - `lessonRawContent`
     - `resourceLink`

8. **Add a Split In Batches node**
   - Node type: `Split In Batches`
   - Name: `Loop Each Subscriber`
   - Connect `Match Subscriber to Lesson -> Loop Each Subscriber`
   - Leave default settings if you want one-item iteration behavior in this pattern.

9. **Add the OpenAI node**
   - Node type: `OpenAI` (LangChain/OpenAI chat node)
   - Name: `Personalize Lesson`
   - Connect the iterative output of `Loop Each Subscriber -> Personalize Lesson`
   - Configure OpenAI credentials with a valid API key
   - Set model to `gpt-4o`
   - Add:
     - a system message instructing the model to act as an expert women’s career coach and return strict JSON only
     - a user message containing:
       - name
       - topic interest
       - learning phase
       - day number
       - lesson title
       - raw content
   - Require output in this structure:
     - `personalizedLesson`
     - `actionableTask`
     - `motivationalQuote`
     - `keyTakeaways` as an array of 3 items

10. **Add a Code node to build the email**
    - Node type: `Code`
    - Name: `Build Personalized Email`
    - Connect `Personalize Lesson -> Build Personalized Email`
    - In the code:
      - extract the AI text from the OpenAI node output
      - strip code fences if present
      - parse JSON
      - add a fallback object if parsing fails
      - merge parsed AI content with subscriber and lesson data
      - build an HTML email template with:
        - header
        - lesson box
        - task box
        - quote box
        - takeaways list
        - CTA button linking to `resourceLink`
    - Return:
      - subscriber metadata
      - parsed AI fields
      - `htmlEmail`

11. **Add a Gmail node**
    - Node type: `Gmail`
    - Name: `Send Daily Lesson Email`
    - Connect `Build Personalized Email -> Send Daily Lesson Email`
    - Configure Gmail OAuth2 credentials
    - Set recipient to `{{$json.email}}`
    - Set message body to `{{$json.htmlEmail}}`
    - Set subject dynamically, for example:
      - `🌟 Day {{$json.dayOfYear}} | {{$json.lessonCategory}}: {{$json.lessonTitle}}`
    - Ensure the body is sent as HTML if your Gmail node version exposes that setting.

12. **Add a final Google Sheets logging node**
    - Node type: `Google Sheets`
    - Name: `Log Delivery to Sheets`
    - Connect `Send Daily Lesson Email -> Log Delivery to Sheets`
    - Configure append operation
    - Select the `SendLog` sheet
    - Map fields manually:
      - `Date` → `{{$now.toISO()}}`
      - `Subscriber Email` → `{{$json.email}}`
      - `Subscriber Name` → `{{$json.name}}`
      - `Lesson Title` → `{{$json.lessonTitle}}`
      - `Category` → `{{$json.lessonCategory}}`
      - `Phase` → `{{$json.phase}}`
      - `Day Number` → `{{$json.dayOfYear}}`
      - `AI Snippet` → `{{$json.personalizedLesson}}`

13. **Close the batch loop**
    - Connect `Log Delivery to Sheets -> Loop Each Subscriber`
    - This allows the next subscriber item to continue through the same AI/email/logging path.

14. **Optionally add sticky notes**
    - Add documentation notes for:
      - workflow overview and setup
      - scheduling + data fetching
      - AI personalization block
      - email sending + logging

15. **Test manually before activating**
    - Run the workflow once manually
    - Confirm:
      - subscribers are loaded
      - phases calculate correctly
      - a lesson is selected
      - OpenAI returns valid JSON
      - HTML email renders correctly
      - Gmail sends successfully
      - SendLog receives appended rows

16. **Fix the issues present in the original implementation before production**
    - Correct the Gmail subject expression by removing extra braces
   - Correct the `Subscriber Name` / `Subscriber Email` logging field mapping
   - Verify item-linking in `Build Personalized Email`; where possible, carry forward all needed fields directly instead of re-fetching them from `Match Subscriber to Lesson`
   - Consider escaping HTML content or sanitizing AI output
   - Consider adding error handling branches for OpenAI, Gmail, and Sheets failures

17. **Activate the workflow**
   - Once credentials, mappings, and test results are confirmed, switch the workflow to active.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Workflow concept: fully personalized daily learning emails for women, tailored by topic interest and time in program. | General design intent |
| Setup prerequisites: OpenAI API access, Google Sheets OAuth2, Gmail OAuth2, and a spreadsheet with `Subscribers`, `ContentLibrary`, and `SendLog`. | Environment requirements |
| The workflow is currently inactive in the provided JSON (`active: false`). | Deployment state |
| The spreadsheet used in the workflow is named `Women Skill Learning Automation`. | Google Sheets context |
| The fallback resource link in matching/email logic is `https://n8n.io`. | Default link |
| The visual notes describe the process as: fetch subscribers and content daily, compute phase, match lesson, personalize with GPT-4o, send branded Gmail email, and log each send. | Embedded workflow documentation |
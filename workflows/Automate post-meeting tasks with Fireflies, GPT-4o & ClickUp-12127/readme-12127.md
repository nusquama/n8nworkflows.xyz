Automate post-meeting tasks with Fireflies, GPT-4o & ClickUp

https://n8nworkflows.xyz/workflows/automate-post-meeting-tasks-with-fireflies--gpt-4o---clickup-12127


# Automate post-meeting tasks with Fireflies, GPT-4o & ClickUp

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** Automate post-meeting operations each morning by pulling Fireflies.ai meeting transcripts, using an OpenAI model to (a) create ClickUp tasks for the meeting host’s action items and (b) draft a follow-up email, which is sent only after Telegram approval.

**Target use cases:**
- Daily “meeting hygiene”: turning transcripts into structured tasks.
- Consistent, fast follow-up emails to external participants with human approval.
- Teams using Fireflies.ai for transcription, ClickUp for task tracking, Telegram for approvals, and Gmail for outbound email.

### Logical blocks
**1.1 Scheduling / Entry**
- Runs daily at 09:00 (cron).

**1.2 Fetch & Filter Meetings (Fireflies)**
- Fetches transcripts since midnight (UTC string), then filters meetings whose `dateString` falls within “today” (local time).

**1.3 Per-meeting Transcript Enrichment**
- Loops over each filtered meeting.
- Fetches the full transcript content and normalizes it into a single object containing transcript text, inferred external participant, and action items.

**1.4 AI Task Extraction → ClickUp Task Creation**
- An AI Agent analyzes transcript + action items and creates ClickUp tasks via a ClickUp tool node.
- Produces a human-readable summary (`$json.output`) of created tasks.

**1.5 Human Approval (Telegram)**
- Sends a Telegram approval message asking whether to send the follow-up email.
- Branches based on approval.

**1.6 AI Email Drafting + Gmail Send + Notifications**
- If approved: drafts a structured JSON email (subject/body/recipient/sender), sends via Gmail, then notifies success in Telegram.
- If rejected: notifies cancellation in Telegram.
- Both branches continue the loop to process remaining meetings.

---

## 2. Block-by-Block Analysis

### 2.1 Scheduling / Entry

**Overview:** Starts the workflow every morning at a fixed time.

**Nodes involved:** `Every Morning`

#### Node: Every Morning
- **Type / role:** Schedule Trigger (`n8n-nodes-base.scheduleTrigger`) — workflow entrypoint.
- **Configuration:** Cron expression `0 9 * * *` (runs daily at 09:00 server time).
- **Inputs/Outputs:** No inputs. Outputs to **Fetch Recent Meetings**.
- **Version notes:** typeVersion `1.2` (standard schedule node).
- **Edge cases / failures:**
  - Timezone depends on n8n instance/server settings; may not match user’s local time expectations.

---

### 2.2 Fetch & Filter Meetings (Fireflies)

**Overview:** Calls Fireflies GraphQL API to list transcripts from today’s date, then filters to those whose `dateString` falls within today (local midnight to tomorrow midnight).

**Nodes involved:** `Fetch Recent Meetings`, `Filter Today's Meetings`

#### Node: Fetch Recent Meetings
- **Type / role:** HTTP Request (`n8n-nodes-base.httpRequest`) — Fireflies GraphQL query.
- **Configuration choices:**
  - **Method:** POST
  - **URL:** `https://api.fireflies.ai/graphql`
  - **Body:** JSON GraphQL request:
    - Query: `transcripts(fromDate: $fromDate) { id title dateString }`
    - Variables: `fromDate` set to `{{ $today.format('YYYY-MM-DD') }}T00:00:00.000Z`
  - **Auth:** `httpHeaderAuth` via generic credential type (expects `Authorization: Bearer <API_KEY>`).
- **Key expressions/vars:**
  - `{{ $today.format('YYYY-MM-DD') }}` produces date string; appended with `T00:00:00.000Z`.
- **Inputs/Outputs:** Input from trigger. Output to **Filter Today's Meetings**.
- **Version notes:** typeVersion `4.1`.
- **Edge cases / failures:**
  - Fireflies auth failures (401) if header auth misconfigured.
  - GraphQL errors returned in `errors` field; node may still succeed but downstream code expects `data.transcripts`.
  - The `fromDate` uses midnight **UTC**, while filtering later uses **local** midnight—can cause off-by-one-day behavior depending on timezone.

#### Node: Filter Today's Meetings
- **Type / role:** Code (`n8n-nodes-base.code`) — filters transcripts to “today”.
- **Configuration choices (interpreted):**
  - Reads `response.data.transcripts`.
  - Builds `today` at local midnight and `tomorrow` at local midnight + 1 day.
  - Keeps meetings where `new Date(meeting.dateString)` is `>= today` and `< tomorrow`.
  - Returns an array of meeting objects directly (n8n treats them as items).
- **Key expressions/vars:** Uses `$input.first().json` and optional chaining `response.data?.transcripts`.
- **Inputs/Outputs:** Input from **Fetch Recent Meetings**. Output to **Loop Over Items**.
- **Version notes:** typeVersion `2`.
- **Edge cases / failures:**
  - If Fireflies returns `dateString` not parseable by `Date()`, filtering will exclude items.
  - If no transcripts exist, returns empty list → loop does nothing (expected).
  - Timezone mismatch (see prior node) can exclude meetings around midnight.

---

### 2.3 Per-meeting Transcript Enrichment

**Overview:** Iterates through meetings one by one, fetches the full transcript details, and formats them into a single normalized object for AI and email steps.

**Nodes involved:** `Loop Over Items`, `Get Full Transcript`, `Format Transcript Data`

#### Node: Loop Over Items
- **Type / role:** Split In Batches (`n8n-nodes-base.splitInBatches`) — iterative processing.
- **Configuration choices:** Uses default options (batch size default in n8n is typically 1 unless specified).
- **Inputs/Outputs:**
  - Input: filtered meetings list.
  - Output (batch): to **Get Full Transcript**.
  - Second output is used to control iteration; the workflow loops back into this node after completion via downstream Telegram notifications.
- **Version notes:** typeVersion `3`.
- **Edge cases / failures:**
  - If batch size defaults unexpectedly, might process multiple meetings per iteration; adjust batch size explicitly if needed.
  - Loop continuation is wired from “Notify Success” and “Notify Cancellation”; if the email branch errors before reaching those nodes, the loop may stall.

#### Node: Get Full Transcript
- **Type / role:** HTTP Request — Fireflies GraphQL query for a specific transcript.
- **Configuration choices:**
  - POST to `https://api.fireflies.ai/graphql`
  - Query: `transcript(id: $id) { id title dateString sentences { text speaker_name } summary { action_items } meeting_attendees { email name } }`
  - Variables: `id` = `{{ $json.id }}`
  - Auth: same `httpHeaderAuth`.
- **Inputs/Outputs:** Input from **Loop Over Items** (each item must include `id`). Output to **Format Transcript Data**.
- **Version notes:** typeVersion `4.1`.
- **Edge cases / failures:**
  - Missing `id` in upstream item breaks query.
  - Large transcripts may increase response size/time.
  - Fireflies may return `null` transcript or partial fields; downstream code handles missing transcript by returning `[]`.

#### Node: Format Transcript Data
- **Type / role:** Code — normalizes transcript into fields used by AI + email.
- **Configuration choices (interpreted):**
  - Reads `item.json.data.transcript`.
  - Concatenates `sentences` into `fullTranscript` lines formatted `speaker_name: text`.
  - Extracts: `title`, `dateString`, `meeting_attendees`, `summary.action_items`.
  - Determines an **external participant**:
    - Uses `hostNames` alias list (CUSTOMIZATION REQUIRED).
    - Picks the first attendee name/email that doesn’t match any host alias.
  - Determines **externalEmail** by picking the first attendee email that does **not** include `yourcompany.com` (CUSTOMIZATION REQUIRED).
  - Outputs a single item with:
    - `title`, `dateString`, `fullTranscript`, `externalParticipant`, `externalEmail`, `actionItems`, and `originalData` (raw transcript object).
- **Key expressions/vars:**
  - Hardcoded arrays/strings requiring edits:
    - `hostNames = ['Host','My Name','Me','TJ']`
    - `yourcompany.com` domain check
- **Inputs/Outputs:** Input from **Get Full Transcript**. Output to **Analyze & Create Tasks**.
- **Version notes:** typeVersion `2`.
- **Edge cases / failures:**
  - If `meeting_attendees` is empty, external participant/email fall back to defaults.
  - Domain filtering can misclassify external vs internal emails if your domain differs or attendees use personal emails.
  - `originalData.host_email` is referenced later in the email prompt, but **this workflow’s Fireflies query does not request `host_email`**. That means the Draft Email step may output an empty sender email unless Fireflies includes it implicitly (unlikely). This is a key integration bug to fix (see reproduction section).

---

### 2.4 AI Task Extraction → ClickUp Task Creation

**Overview:** AI agent analyzes the meeting transcript and action items, and creates ClickUp tasks for the host using a ClickUp tool. Produces a summary for approval.

**Nodes involved:** `Analyze & Create Tasks`, `OpenAI Chat Model`, `Think`, `Create ClickUp Task`

#### Node: OpenAI Chat Model
- **Type / role:** LangChain OpenAI Chat model (`@n8n/n8n-nodes-langchain.lmChatOpenAi`) — LLM backend for the agent.
- **Configuration choices:** Model `openai/gpt-4o-mini`.
- **Inputs/Outputs:** Connected as `ai_languageModel` into **Analyze & Create Tasks**.
- **Version notes:** typeVersion `1.2`.
- **Edge cases / failures:**
  - Missing/invalid OpenAI credentials.
  - Rate limits / model availability.

#### Node: Think
- **Type / role:** LangChain “Think Tool” (`@n8n/n8n-nodes-langchain.toolThink`) — allows the agent to internally plan.
- **Inputs/Outputs:** Exposed as an `ai_tool` to **Analyze & Create Tasks**.
- **Version notes:** typeVersion `1.1`.
- **Edge cases / failures:** Generally none; it’s an internal reasoning tool.

#### Node: Create ClickUp Task
- **Type / role:** ClickUp Tool (`n8n-nodes-base.clickUpTool`) — tool the agent calls to create tasks.
- **Configuration choices:**
  - **Auth:** OAuth2.
  - **Fields mapped from AI tool inputs** using `$fromAI(...)`:
    - Task name: `task_name`
    - Content/description: `Content`
    - Due date: `Due_Date` (string)
    - Priority: `Priority` (number)
- **Inputs/Outputs:** Provided as `ai_tool` to **Analyze & Create Tasks** (agent invokes it).
- **Version notes:** typeVersion `1`.
- **Edge cases / failures:**
  - ClickUp OAuth token expired/invalid.
  - Missing required ClickUp context (Workspace/Space/List) depending on how the credential/tool is set up in your n8n; tool may require a default List preconfigured in node settings (not shown in the snippet).
  - AI may output malformed due date or priority outside ClickUp accepted values; creation can fail.

#### Node: Analyze & Create Tasks
- **Type / role:** LangChain Agent (`@n8n/n8n-nodes-langchain.agent`) — orchestrates reasoning + tool calls.
- **Configuration choices (interpreted):**
  - Prompt instructs:
    - Identify host-owned action items.
    - Create ClickUp tasks with priority, context, due dates when mentioned.
    - Only create tasks clearly assigned to host.
    - Output a summary of tasks created.
  - Uses:
    - OpenAI Chat Model as LLM.
    - Think tool.
    - ClickUp task tool.
  - Has output parser enabled (`hasOutputParser: true`), but no explicit output parser node connected for this agent (it may use internal parsing).
- **Key expressions/vars:** Injects `{{$json.title}}`, `{{$json.dateString}}`, `{{$json.fullTranscript}}`, `{{$json.actionItems}}`, `{{$now}}`.
- **Inputs/Outputs:** Input from **Format Transcript Data**. Output to **Ask Approval (Telegram)**.
- **Version notes:** typeVersion `2.2`.
- **Edge cases / failures:**
  - If transcript is huge, token limits may truncate content; consider summarization or chunking.
  - Agent might create duplicate tasks if rerun; no idempotency checks exist.
  - If `actionItems` is empty, agent relies entirely on transcript parsing.

---

### 2.5 Human Approval (Telegram)

**Overview:** Sends a Telegram approval request containing the AI summary; continues to email drafting only if approved.

**Nodes involved:** `Ask Approval (Telegram)`, `Approved?`

#### Node: Ask Approval (Telegram)
- **Type / role:** Telegram (`n8n-nodes-base.telegram`) — send-and-wait approval.
- **Configuration choices:**
  - Operation: `sendAndWait` with `approvalType: double` (two-button approval).
  - Message includes `{{ $json.output }}` from the prior AI agent output.
  - Attribution disabled.
- **Inputs/Outputs:** Input from **Analyze & Create Tasks**. Output to **Approved?**.
- **Version notes:** typeVersion `1.2`.
- **Edge cases / failures:**
  - Telegram credentials/chat ID not configured.
  - “Send and wait” requires webhook/polling configuration in n8n; may time out depending on instance settings.
  - If `$json.output` is missing, message may be unclear.

#### Node: Approved?
- **Type / role:** IF (`n8n-nodes-base.if`) — approval gate.
- **Configuration choices:**
  - Condition checks whether `{{ $json.data }}` **contains** `"true"`.
  - True branch → **Draft Email**
  - False branch → **Notify Cancellation**
- **Inputs/Outputs:** Input from Telegram approval result. Two outputs: approved and rejected.
- **Version notes:** typeVersion `2.2`.
- **Edge cases / failures:**
  - Telegram approval payload structure can vary; relying on string “contains true” is brittle.
  - If `data` is boolean instead of string, “contains” may behave unexpectedly under loose typing.

---

### 2.6 Email Automation (Draft → Send → Notify → Continue Loop)

**Overview:** If approved, drafts a structured follow-up email with an AI agent, parses it as JSON, sends via Gmail, notifies success. If not approved, notifies cancellation. Both paths rejoin by continuing the loop.

**Nodes involved:** `Draft Email`, `OpenAI Chat Model1`, `Structured Output Parser`, `Send Gmail`, `Notify Success`, `Notify Cancellation`

#### Node: OpenAI Chat Model1
- **Type / role:** OpenAI chat model for the email agent.
- **Configuration:** Model `openai/gpt-4o-mini`.
- **Inputs/Outputs:** Connected as `ai_languageModel` to **Draft Email**.
- **Version notes:** typeVersion `1.2`.
- **Edge cases / failures:** Same as the other model node (auth/rate limits).

#### Node: Structured Output Parser
- **Type / role:** Structured output parser (`@n8n/n8n-nodes-langchain.outputParserStructured`) — enforces JSON shape.
- **Configuration choices:** Schema example includes:
  - `subject`, `body`, `recipient_email`
  - (Note: `sender_email` is requested in prompt but not in schema example.)
- **Inputs/Outputs:** Connected as `ai_outputParser` to **Draft Email**.
- **Version notes:** typeVersion `1.3`.
- **Edge cases / failures:**
  - If the model returns non-JSON or invalid JSON, parsing fails and stops the approved branch.
  - Schema mismatch: prompt demands `sender_email`, but parser schema example doesn’t include it; depending on parser strictness, it may drop or reject it.

#### Node: Draft Email
- **Type / role:** LangChain Agent — generates JSON email (subject/body/recipient/sender).
- **Configuration choices:**
  - Prompt references data from **Format Transcript Data** using cross-node expressions:
    - Title/date/externalParticipant/externalEmail/fullTranscript/actionItems.
    - Sender/To fields attempt to use:
      - `originalData.host_email`
      - `originalData.meeting_attendees[1].email`
      - `originalData.meeting_attendees[0].email`
  - Requires output in exact JSON.
  - Uses Structured Output Parser.
- **Inputs/Outputs:** Input from **Approved?** true branch. Output to **Send Gmail**.
- **Version notes:** typeVersion `2.2`.
- **Edge cases / failures:**
  - **Likely bug:** `originalData.host_email` is not fetched by Fireflies query; will be undefined.
  - Indexing `meeting_attendees[1]` can fail if fewer than 2 attendees.
  - If transcript is long, model may exceed limits or ignore strict JSON instruction.

#### Node: Send Gmail
- **Type / role:** Gmail (`n8n-nodes-base.gmail`) — sends the follow-up email.
- **Configuration choices:**
  - To: `{{ $json.output.recipient_email }}`
  - Subject: `{{ $json.output.subject }}`
  - Message: `{{ $json.output.body }}`
  - Email type: text, attribution disabled.
- **Inputs/Outputs:** Input from **Draft Email**. Output to **Notify Success**.
- **Version notes:** typeVersion `2.1`.
- **Edge cases / failures:**
  - Gmail OAuth not configured or token expired.
  - If `recipient_email` is invalid/empty, send fails.
  - If you intend to send from a specific mailbox, ensure the Gmail credential matches that sender.

#### Node: Notify Success
- **Type / role:** Telegram — success notification.
- **Configuration choices:** Sends: `I Have sent the person message out at this adress {{ $('Draft Email').item.json.output.recipient_email }}`
- **Inputs/Outputs:** Input from **Send Gmail**. Output back to **Loop Over Items** (continues processing next meeting).
- **Version notes:** typeVersion `1.2`. `retryOnFail: true`.
- **Edge cases / failures:**
  - Message text has minor grammar issues (cosmetic).
  - If Draft Email output missing, expression resolves empty.

#### Node: Notify Cancellation
- **Type / role:** Telegram — cancellation notification.
- **Configuration choices:** Text: “Okay I will not send the email out”
- **Inputs/Outputs:** Input from **Approved?** false branch. Output back to **Loop Over Items**.
- **Version notes:** typeVersion `1.2`.
- **Edge cases / failures:** Telegram credential/chat issues prevent loop continuation unless handled.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Every Morning | Schedule Trigger | Daily entrypoint at 09:00 | — | Fetch Recent Meetings | ## 1. Fetch & Filter Meetings<br>Get transcripts from Fireflies and filter for today's date. |
| Fetch Recent Meetings | HTTP Request | Query Fireflies transcripts since midnight | Every Morning | Filter Today's Meetings | ## 1. Fetch & Filter Meetings<br>Get transcripts from Fireflies and filter for today's date. |
| Filter Today's Meetings | Code | Keep only meetings occurring “today” | Fetch Recent Meetings | Loop Over Items | ## 1. Fetch & Filter Meetings<br>Get transcripts from Fireflies and filter for today's date. |
| Loop Over Items | Split In Batches | Iterate meetings one-by-one | Filter Today's Meetings; Notify Success; Notify Cancellation | Get Full Transcript (batch output) | ## 1. Fetch & Filter Meetings<br>Get transcripts from Fireflies and filter for today's date. |
| Get Full Transcript | HTTP Request | Fetch full transcript details | Loop Over Items | Format Transcript Data | ## 1. Fetch & Filter Meetings<br>Get transcripts from Fireflies and filter for today's date. |
| Format Transcript Data | Code | Build full transcript text + infer external participant | Get Full Transcript | Analyze & Create Tasks | ## 1. Fetch & Filter Meetings<br>Get transcripts from Fireflies and filter for today's date. |
| Analyze & Create Tasks | LangChain Agent | Extract host action items and create ClickUp tasks | Format Transcript Data | Ask Approval (Telegram) | ## 2. Analysis & Task Management<br>AI extracts action items and syncs them to ClickUp. |
| OpenAI Chat Model | OpenAI Chat Model (LangChain) | LLM for task agent | — | Analyze & Create Tasks (ai_languageModel) | ## 2. Analysis & Task Management<br>AI extracts action items and syncs them to ClickUp. |
| Think | Think Tool (LangChain) | Planning tool for agent | — | Analyze & Create Tasks (ai_tool) | ## 2. Analysis & Task Management<br>AI extracts action items and syncs them to ClickUp. |
| Create ClickUp Task | ClickUp Tool | Tool called by agent to create ClickUp tasks | — | Analyze & Create Tasks (ai_tool) | ## 2. Analysis & Task Management<br>AI extracts action items and syncs them to ClickUp. |
| Ask Approval (Telegram) | Telegram | Human approval request (send-and-wait) | Analyze & Create Tasks | Approved? | ## 3. Human Approval<br>Telegram bot asks for confirmation before sending emails. |
| Approved? | IF | Branch on approval | Ask Approval (Telegram) | Draft Email; Notify Cancellation | ## 3. Human Approval<br>Telegram bot asks for confirmation before sending emails. |
| Draft Email | LangChain Agent | Generate JSON follow-up email | Approved? (true) | Send Gmail | ## 4. Email Automation<br>Drafts and sends follow-ups via Gmail if approved. |
| OpenAI Chat Model1 | OpenAI Chat Model (LangChain) | LLM for email agent | — | Draft Email (ai_languageModel) | ## 4. Email Automation<br>Drafts and sends follow-ups via Gmail if approved. |
| Structured Output Parser | Structured Output Parser (LangChain) | Enforce structured JSON output for email | — | Draft Email (ai_outputParser) | ## 4. Email Automation<br>Drafts and sends follow-ups via Gmail if approved. |
| Send Gmail | Gmail | Send the drafted email | Draft Email | Notify Success | ## 4. Email Automation<br>Drafts and sends follow-ups via Gmail if approved. |
| Notify Success | Telegram | Notify email sent + continue loop | Send Gmail | Loop Over Items | ## 4. Email Automation<br>Drafts and sends follow-ups via Gmail if approved. |
| Notify Cancellation | Telegram | Notify not sending + continue loop | Approved? (false) | Loop Over Items | ## 3. Human Approval<br>Telegram bot asks for confirmation before sending emails. |
| Overview | Sticky Note | Documentation / setup notes | — | — | ### Meeting Intelligence Automation<br>This workflow automates your post-meeting routine by analyzing transcripts to create tasks and draft follow-up emails.<br><br>### How it works<br>1. **Fetch:** Retrieves meeting transcripts via the Fireflies.ai API.<br>2. **Analyze:** AI processes the transcript to extract action items assigned to you.<br>3. **Task Creation:** Creates tasks in ClickUp automatically.<br>4. **Approval:** Sends a Telegram message asking if you want to send a follow-up email.<br>5. **Email:** If approved, drafts and sends a personalized summary email via Gmail.<br><br>### Setup Steps<br>1. **Credentials:** <br>   - Get your API Key from Fireflies.ai (Settings -> Integrations -> Fireflies API).<br>   - Create a "Header Auth" credential in n8n named `Authorization` with value `Bearer <YOUR_KEY>`.<br>2. **Configuration:** <br>   - In the 'Format Transcript Data' code node, update the `hostNames` array with your name.<br>   - In the Telegram nodes, input your Chat ID. |
| Section 1 | Sticky Note | Visual section label | — | — | ## 1. Fetch & Filter Meetings<br>Get transcripts from Fireflies and filter for today's date. |
| Section 2 | Sticky Note | Visual section label | — | — | ## 2. Analysis & Task Management<br>AI extracts action items and syncs them to ClickUp. |
| Section 3 | Sticky Note | Visual section label | — | — | ## 3. Human Approval<br>Telegram bot asks for confirmation before sending emails. |
| Section 4 | Sticky Note | Visual section label | — | — | ## 4. Email Automation<br>Drafts and sends follow-ups via Gmail if approved. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Trigger**
1. Add **Schedule Trigger** node named **Every Morning**.
2. Set cron to `0 9 * * *` (adjust timezone in n8n settings if needed).
3. Connect to the next node.

2) **Configure Fireflies credential**
4. Create an **HTTP Header Auth** credential.
   - Header name: `Authorization`
   - Value: `Bearer <YOUR_FIREFLIES_API_KEY>`
5. (Optional) Test with a simple Fireflies API call.

3) **Fetch transcript list**
6. Add **HTTP Request** node **Fetch Recent Meetings**.
7. Method: POST; URL: `https://api.fireflies.ai/graphql`.
8. Authentication: **Generic Credential Type → HTTP Header Auth** (select your credential).
9. Body type: JSON and enable “Send Body”.
10. Use this GraphQL payload (adapt only the date expression if desired):
   - Query: `transcripts(fromDate: $fromDate) { id title dateString }`
   - Variables: `fromDate = {{$today.format('YYYY-MM-DD')}}T00:00:00.000Z`
11. Connect **Every Morning → Fetch Recent Meetings**.

4) **Filter to today**
12. Add **Code** node **Filter Today's Meetings**.
13. Paste logic that reads `data.transcripts`, computes local `today`/`tomorrow`, and filters by `dateString`.
14. Connect **Fetch Recent Meetings → Filter Today's Meetings**.

5) **Loop meetings**
15. Add **Split In Batches** node **Loop Over Items**.
16. Set **Batch Size = 1** explicitly (recommended for predictable behavior).
17. Connect **Filter Today's Meetings → Loop Over Items**.

6) **Fetch full transcript**
18. Add **HTTP Request** node **Get Full Transcript**.
19. Same Fireflies URL/auth as before.
20. GraphQL query should request the fields you will use later. Recommended to fix the missing field issue by including `host_email` if Fireflies supports it; if it doesn’t, remove its usage later:
   - `transcript(id: $id) { id title dateString sentences { text speaker_name } summary { action_items } meeting_attendees { email name } host_email }`
21. Variable `id = {{$json.id}}`.
22. Connect **Loop Over Items (batch output) → Get Full Transcript**.

7) **Format transcript data**
23. Add **Code** node **Format Transcript Data**.
24. Implement:
   - Build `fullTranscript` from sentences.
   - Extract `title`, `dateString`, `actionItems`.
   - Set `hostNames` to your real name/aliases.
   - Replace `yourcompany.com` with your domain.
25. Output one item with fields: `title`, `dateString`, `fullTranscript`, `externalParticipant`, `externalEmail`, `actionItems`, `originalData`.
26. Connect **Get Full Transcript → Format Transcript Data**.

8) **Set up ClickUp tool**
27. Add **ClickUp Tool** node **Create ClickUp Task**.
28. Configure ClickUp **OAuth2** credential.
29. Ensure the node is configured to create tasks in the correct ClickUp List/Space (depends on node UI; select the target List).
30. Map fields from AI:
   - Name: `{{$fromAI("task_name")}}`
   - Content: `{{$fromAI("Content")}}`
   - Due date: `{{$fromAI("Due_Date")}}`
   - Priority: `{{$fromAI("Priority")}}`

9) **Set up AI agent for task creation**
31. Add **OpenAI Chat Model** node (LangChain) and select `openai/gpt-4o-mini` (configure OpenAI credentials).
32. Add **Think Tool** node.
33. Add **AI Agent** node **Analyze & Create Tasks** with instructions to:
   - Identify host-owned action items
   - Call “Create ClickUp Task” tool for each
   - Produce a summary output
34. Connect:
   - **OpenAI Chat Model → Analyze & Create Tasks** (ai_languageModel)
   - **Think → Analyze & Create Tasks** (ai_tool)
   - **Create ClickUp Task → Analyze & Create Tasks** (ai_tool)
35. Connect **Format Transcript Data → Analyze & Create Tasks**.

10) **Telegram approval**
36. Create Telegram credentials (BotFather token) and ensure chat ID is known.
37. Add Telegram node **Ask Approval (Telegram)** with operation **sendAndWait** and double-approval buttons.
38. Message: include `{{$json.output}}`.
39. Connect **Analyze & Create Tasks → Ask Approval (Telegram)**.

11) **Approval branching**
40. Add **IF** node **Approved?**.
41. Condition: check approval payload (recommended improvement: check a boolean field rather than “contains true”).
42. Connect **Ask Approval (Telegram) → Approved?**.

12) **Email drafting agent**
43. Add **Structured Output Parser** and define a schema that includes **subject/body/recipient_email/sender_email** (align parser with prompt).
44. Add **OpenAI Chat Model1** (same model).
45. Add **AI Agent** node **Draft Email**:
   - Prompt requires exact JSON output.
   - Use values from **Format Transcript Data** (title/date/transcript/action items/external email).
   - Avoid unsafe indexing like `meeting_attendees[1]` unless you guard it in code first.
46. Connect:
   - **OpenAI Chat Model1 → Draft Email** (ai_languageModel)
   - **Structured Output Parser → Draft Email** (ai_outputParser)
47. Connect **Approved? (true) → Draft Email**.

13) **Send email via Gmail**
48. Add **Gmail** node **Send Gmail** with OAuth2 credentials.
49. Set:
   - To: `{{$json.output.recipient_email}}`
   - Subject: `{{$json.output.subject}}`
   - Body: `{{$json.output.body}}` (text)
50. Connect **Draft Email → Send Gmail**.

14) **Notifications + loop continuation**
51. Add Telegram node **Notify Success**, connect **Send Gmail → Notify Success**.
52. Add Telegram node **Notify Cancellation**, connect **Approved? (false) → Notify Cancellation**.
53. Connect **Notify Success → Loop Over Items** and **Notify Cancellation → Loop Over Items** to continue processing remaining meetings.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Meeting Intelligence Automation” overview and setup guidance | Provided in the workflow’s Overview sticky note (credentials + configuration reminders). |
| Fireflies API key location: Settings → Integrations → Fireflies API | Mentioned in sticky note. |
| n8n Header Auth should set `Authorization: Bearer <YOUR_KEY>` | Mentioned in sticky note. |
| Customize `hostNames` and internal email domain filtering in “Format Transcript Data” | Mentioned in sticky note; required for correct external participant detection. |
| Important integration gap: `host_email` is referenced later but not fetched in the Fireflies GraphQL query | Fix by adding `host_email` to the query (if supported) or removing/deriving sender email another way. |
| Output parser schema should include `sender_email` if the prompt demands it | Align schema and prompt to prevent parsing failures. |
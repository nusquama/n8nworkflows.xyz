Route and categorize Gmail emails to Slack with Llama 3 via OpenRouter

https://n8nworkflows.xyz/workflows/route-and-categorize-gmail-emails-to-slack-with-llama-3-via-openrouter-12671


# Route and categorize Gmail emails to Slack with Llama 3 via OpenRouter

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Route and categorize Gmail emails to Slack with Llama 3 via OpenRouter  
**Workflow name (n8n):** Gmail-to-Slack Email Automation

**Purpose:**  
Automatically monitor **unread Gmail emails**, **filter/deduplicate** them, categorize each email using **Llama 3 via OpenRouter**, then route the email into a matching **Slack channel**. If a matching channel does not exist, the workflow creates it, invites a user, and posts the email.

**Target use cases:**
- Team inbox triage into Slack (Sales/Marketing/Product/Engineering/etc.)
- Auto-routing inbound leads/support/product feedback into dedicated channels
- Creating new Slack channels for new categories discovered by the model

### Logical blocks
1.1 **Gmail Intake & Initial Filtering**  
Trigger on unread Gmail, deduplicate by Message-ID, block disposable/free domains, stop flow if empty.

1.2 **Email Data Preparation**  
Select only relevant email fields to reduce payload.

1.3 **AI Categorization (OpenRouter + Structured JSON)**  
Use an AI Agent with a strict JSON schema output parser to produce `{ "category": "..." }`.

1.4 **Merge Category into Email Payload**  
Combine AI output with the original email fields.

1.5 **Slack Channel Lookup & Match**  
Fetch all Slack channels, normalize/format list, then find a channel whose name matches/includes the category.

1.6 **Routing: Existing Channel vs New Channel**  
- If channel exists: join channel then post message  
- If not: create channel, invite a user, then post message

---

## 2. Block-by-Block Analysis

### 2.1 Gmail Intake & Initial Filtering
**Overview:** Polls Gmail every minute for unread emails (excluding spam/drafts), then deduplicates and filters senders by domain. Prevents re-processing the same email and blocks unwanted sources.

**Nodes involved:**
- **Capture Gmail Event**
- **Filter + Deduplicate Validation**
- **Check Data Exist**

#### Node: Capture Gmail Event
- **Type / role:** Gmail Trigger — entry point for new unread emails.
- **Configuration (interpreted):**
  - Polling schedule: **every minute**
  - Filters: `readStatus = unread`, `includeDrafts = false`, `includeSpamTrash = false`
  - Downloads attachments: **enabled**
  - `simple = false` (returns richer payload including headers/metadata)
- **Outputs:** Email item(s) including headers, from/to, subject, body, id/threadId, attachments (depending on Gmail payload).
- **Credentials:** Gmail OAuth2 (“Gmail account 3”).
- **Failure modes / edge cases:**
  - OAuth token expiration/permissions missing (401/403)
  - Gmail API quota limits
  - Attachments can increase payload size; large attachments may impact execution time/memory.

#### Node: Filter + Deduplicate Validation
- **Type / role:** Code node (run once per item) — dedupe + domain filtering.
- **Key logic / variables:**
  - Uses `const data = $getWorkflowStaticData('global')` to persist a `processed` map across executions.
  - Deduplication key: `headers['message-id']`
  - If already processed: `return {};` (emits an empty object)
  - Domain filters:
    - Blocks known disposable domains (yopmail, mailinator, etc.)
    - Blocks free email providers (gmail.com, yahoo.com, outlook.com, hotmail.com, icloud.com) — marked “optional” but enforced in code.
- **Inputs:** Gmail trigger item (`$json` containing headers/from).
- **Outputs:** Either the original item (`return $input.item`) or an empty object `{}`.
- **Failure modes / edge cases:**
  - If `headers['message-id']` is missing or undefined, dedupe may behave unexpectedly (all such emails share the same `undefined` key). Consider adding a fallback (e.g., Gmail `id`).
  - Blocking free providers may unintentionally block legitimate leads/customers using Gmail.
  - Workflow static data can grow indefinitely; consider TTL/cleanup.

#### Node: Check Data Exist
- **Type / role:** IF node — gate that stops the flow when the previous node returned empty JSON.
- **Condition:** `$json` **not empty** (strict object notEmpty check).
- **True path:** continues to “Arrange Email Data”
- **False path:** stops (no downstream connections).
- **Failure modes / edge cases:**
  - If upstream returned `{}` explicitly, it is “empty” and stops as intended.
  - If upstream returns partial objects, flow continues (ensure domain/dedupe logic returns `{}` for blocked cases as designed).

**Sticky note covering this block:**  
“## Gmail Intake & Data Preparation …” (applies to the Gmail trigger, filter code, IF gate, and field selection node)

---

### 2.2 Email Data Preparation
**Overview:** Reduces the email payload to only the fields needed for classification and Slack posting.

**Nodes involved:**
- **Arrange Email Data**

#### Node: Arrange Email Data
- **Type / role:** Set node — keep selected fields only.
- **Configuration (interpreted):**
  - Include mode: **selected**
  - Kept fields: `id, text, textAsHtml, subject, date, to, from, threadId`
- **Input:** Email item from IF true path.
- **Output:** Cleaned email JSON with only selected fields.
- **Failure modes / edge cases:**
  - If Gmail payload does not contain `text` or `textAsHtml`, later nodes referencing them may error or post empty content.
  - `from` structure can vary; later Slack blocks reference `from.value[0].address` (structure mismatch risk).

---

### 2.3 AI Categorization (OpenRouter + Structured JSON)
**Overview:** Uses a LangChain AI Agent to classify the email into a predefined set of categories (or propose a new one), enforcing strict JSON output.

**Nodes involved:**
- **Find Category**
- **OpenRouter Chat Model**
- **Structured Output Parser**
- **OpenRouter Chat Model1** (present but functionally miswired / unused for parsing in a typical setup; see details)

#### Node: OpenRouter Chat Model
- **Type / role:** LangChain Chat Model (OpenRouter) — LLM used by the agent.
- **Configuration:** model = `meta-llama/llama-3-70b-instruct`
- **Credential:** OpenRouter API (“OpenRouter account 2”)
- **Connection:** Provides `ai_languageModel` input to **Find Category**.
- **Failure modes:**
  - OpenRouter auth errors, insufficient credits, rate limits
  - Model latency/timeouts for large emails

#### Node: Structured Output Parser
- **Type / role:** LangChain Structured Output Parser — enforces JSON schema.
- **Configuration:**
  - `autoFix = true` (attempts to repair malformed JSON)
  - JSON schema example:
    ```json
    { "category": "sales" }
    ```
- **Connection:** Provides `ai_outputParser` input to **Find Category**.
- **Failure modes / edge cases:**
  - If the model outputs non-JSON or additional text, parser may fail; `autoFix` helps but is not guaranteed.
  - Schema is minimal; it does not validate category constraints beyond presence of `category`.

#### Node: OpenRouter Chat Model1
- **Type / role:** LangChain Chat Model (OpenRouter)
- **Configuration:** model = `meta-llama/llama-3-70b-instruct`
- **Credential:** OpenRouter API (“OpenRouter account 2”)
- **Connections:** In this workflow, it is connected as `ai_languageModel` to **Structured Output Parser**.
- **Important note:** Normally, the **Output Parser** is not driven by a language model; it parses the output of an LLM/agent. This extra model node is likely unnecessary or a wiring artifact. The agent already uses **OpenRouter Chat Model** directly.

#### Node: Find Category
- **Type / role:** LangChain Agent — runs the categorization prompt and returns structured output.
- **Prompt behavior (key rules):**
  - Predefined categories: sales, marketing, product, engineering, ai, internal, accounts
  - Special rule: security/login/OTP/2FA/password resets must be **internal**, not accounts
  - If none match: create a **new lowercase, hyphen-separated** category (max 1 word, max 30 chars)
  - Injects email subject/body:
    - `{{$json.subject}}`
    - `{{$json.text}}`
  - Must return ONLY valid JSON: `{ "category": "<category>" }`
- **Inputs:**
  - Main input: from **Arrange Email Data**
  - `ai_languageModel`: from **OpenRouter Chat Model**
  - `ai_outputParser`: from **Structured Output Parser**
- **Output:** An object containing `output.category` (later referenced as `$json.output.category`).
- **Failure modes / edge cases:**
  - Email body may exceed context; consider truncating before sending to model.
  - If `text` is empty, model may classify poorly; fallback to `textAsHtml` stripping could help.
  - New category constraints are prompt-only; Slack channel naming has stricter rules (see Slack block).

**Sticky note covering this block:**  
“## AI Email Categorization …” (applies to Find Category + model usage)

---

### 2.4 Merge Category into Email Payload
**Overview:** Combines the AI category result with the original email fields so downstream Slack logic can use a single consolidated item.

**Nodes involved:**
- **Merge Category**

#### Node: Merge Category
- **Type / role:** Set node — maps/merges fields from two sources using node-item references.
- **Key assignments:**
  - `output.category = {{ $json.output.category }}` (from the AI agent output item)
  - Copies these fields from **Arrange Email Data** via item reference:
    - `id, text, textAsHtml, subject, date, to, from, threadId`
- **Inputs:** From **Find Category**
- **Output:** One item containing email fields + `output.category`.
- **Failure modes / edge cases:**
  - Uses `$('Arrange Email Data').item.json...` references; if items are not aligned (e.g., multiple emails processed in a batch), item mapping may mismatch. Consider using Merge node or ensure 1:1 flow.
  - `output.category` is nested under `output`; later nodes consistently reference `$('Merge Category').first().json.output.category`.

**Sticky note covering this block:**  
“## Merge Node ( Merge Category) …”

---

### 2.5 Slack Channel Lookup & Match
**Overview:** Fetches all Slack channels, formats them, removes duplicates, and attempts to find a channel name that matches the category.

**Nodes involved:**
- **Get Channels List**
- **Format Channel Data**
- **Remove Duplicate Channel + Find Matched Channel**
- **Check Channel Exist**

#### Node: Get Channels List
- **Type / role:** Slack node — retrieves channels.
- **Operation:** `channel.getAll`
- **Credentials:** Slack API (“Slack account”)
- **Output:** List of channel objects (name, id, etc.)
- **Failure modes / edge cases:**
  - Slack app scopes missing (e.g., `channels:read`, `groups:read` depending on public/private)
  - Large workspaces: pagination/volume issues (node handles, but slower)

#### Node: Format Channel Data
- **Type / role:** Code node — normalize Slack channel list to `{channel_name, channel_id}` and de-duplicate.
- **Logic:**
  - Maps each input item to:
    - `channel_name: item.json.name`
    - `channel_id: item.json.id`
  - Filters duplicates by `channel_id`
- **Failure modes / edge cases:**
  - If Slack “Get All” output structure differs (e.g., nested arrays), mapping may fail.
  - Assumes each input item is a single channel.

#### Node: Remove Duplicate Channel + Find Matched Channel
- **Type / role:** Code node — finds best match based on category substring checks.
- **Key variables:**
  - `category = $('Merge Category').first().json.output.category`
  - Matching rule:
    - `channel_name.includes(category)` OR `category.includes(channel_name)`
- **Output:**
  - If match found:
    - `{ found: true, matchedChannelName, matchedChannelId }`
  - Else:
    - `{ found: false, matchedChannelName: null, matchedChannelId: null }`
- **Failure modes / edge cases:**
  - Substring matching can produce false positives (e.g., category `ai` matches many channel names containing “ai”).
  - Does not enforce Slack channel naming constraints for category (only used later for creating).
  - Uses `first()` from Merge Category; if multiple emails processed concurrently, it may mismatch.

#### Node: Check Channel Exist
- **Type / role:** IF node — routes execution based on `found`.
- **Condition:** `{{ $json.found }} == true`
- **True path:** Existing channel flow → Join Slack Channel → Post Message in Channel  
- **False path:** New channel flow → Create Slack Channel → Invite → Post
- **Failure modes:**
  - If previous node returns non-boolean `found`, loose type validation may still pass; ensure consistent output.

**Sticky notes covering this block:**  
- “## Slack Channels …” (channel list + formatting + matching nodes)  
- “## Channel Existence Check …” (the IF router)

---

### 2.6 Routing & Posting to Slack (Existing vs New Channel)
**Overview:** If channel exists, ensure the bot joins and post message. Otherwise create the channel, invite a user, and post the same Block Kit message.

**Nodes involved (existing channel path):**
- **Join Slack Channel**
- **Post Message in Channel**

**Nodes involved (new channel path):**
- **Create Slack Channel**
- **Invite a user to a channel**
- **Post Message in Created Channel**

#### Node: Join Slack Channel
- **Type / role:** Slack node — joins the bot/app to an existing channel.
- **Operation:** `channel.join`
- **Channel ID:** `={{ $json.matchedChannelId }}`
- **Credentials:** Slack API (“Slack account”)
- **Failure modes / edge cases:**
  - Joining private channels requires different scopes and invitation; `join` may fail.
  - If already joined, Slack may return an error or a benign response depending on API behavior.

#### Node: Post Message in Channel
- **Type / role:** Slack node — posts a Block Kit message to the matched channel.
- **Operation:** post message (block format)
- **Channel ID:** `={{ $json.id }}` (note: this `$json` is output of Join Slack Channel; it must include `id`)
- **Message content:**
  - Header text: “📧 New Email Received in {{ $json.name }} Channel”
  - Block Kit includes:
    - Section with From/Subject/snippet
    - Button “Reply via Email” with:
      - `action_id: reply_email`
      - `value: {{ threadId }}`
    - Gmail deep link: `https://mail.google.com/mail/u/0/#inbox/{{id}}`
  - Uses data from `$('Merge Category').first()`:
    - From: `from.value[0].address`
    - Subject: `subject`
    - Body snippet: `text.slice(0, 500)` plus sanitization
- **Failure modes / edge cases:**
  - `from.value[0].address` may not exist depending on Gmail node structure; might need `from.email` or different path.
  - Slack Block Kit has limits (text length, blocks size).
  - The snippet sanitization escapes quotes; still ensure the overall JSON remains valid.
  - If `Join Slack Channel` output doesn’t contain `name` or `id`, channel selection may fail.

#### Node: Create Slack Channel
- **Type / role:** Slack node — creates a new channel.
- **Configuration observed:**
  - `resource = channel`
  - **Potential misconfiguration:** the parameter used is `channelId = {{ $('Merge Category').first().json.output.category }}`. For “create channel”, Slack typically requires a **channel name**, not an ID. In n8n Slack node, this is usually a “name” field. Verify in UI that this node is actually set to “Create” and using the correct parameter.
- **Failure modes / edge cases:**
  - Slack channel naming rules: lowercase, no spaces, limited characters, length constraints. The prompt tries to enforce this but is not guaranteed.
  - Creating a channel that already exists will error.
  - Missing scopes: `channels:manage` (or equivalent) required.

#### Node: Invite a user to a channel
- **Type / role:** Slack node — invites a specific user to the created channel.
- **Operation:** `channel.invite`
- **User IDs:** `["U0A6ULM7CGK"]` (hardcoded)
- **Channel ID:** `={{ $json.id }}` (expects Create Slack Channel output includes channel id)
- **Failure modes / edge cases:**
  - Inviting users requires proper scopes (`channels:manage` / `users:read` depending)
  - If user already in channel, Slack may error or ignore.

#### Node: Post Message in Created Channel
- **Type / role:** Slack node — posts the same Block Kit message to the newly created channel.
- **Channel ID:** `={{ $json.id }}` (expects output from invite node retains channel id; in practice invite response may not include `id`—this can break. Often safer to reference the channel id from Create Slack Channel node explicitly.)
- **Message payload:** Essentially identical to “Post Message in Channel”.
- **Failure modes / edge cases:**
  - If the Invite node does not output `id`, message sending will fail. Prefer:
    - `channelId = {{ $('Create Slack Channel').item.json.id }}` (or equivalent)
  - Same data-path risks as above for `from.value[0].address`.

**Sticky notes covering this block:**  
- “## Existing Channel Flow …” (join + post)  
- “## New Channel Creation Flow …” (create + invite + post)

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Capture Gmail Event | gmailTrigger | Poll unread Gmail emails | — | Filter + Deduplicate Validation | ##  Gmail Intake & Data Preparation<br>* The Gmail node runs every minute to check for newly received emails that are still in an unread state. It is configured to ignore spam and draft emails.<br>* The Code node filters emails based on email IDs. It checks whether the email is from trusted sources and whether an unread email notification has already been sent to the Slack channel. If a notification was already sent, it passes an empty JSON to the next node.<br>* The If node checks whether the incoming data is empty. Only if data is present does the workflow proceed further.<br>* The Edit Fields node removes unnecessary data from the response and retains only the required fields, allowing the AI node to process the data more efficiently. |
| Filter + Deduplicate Validation | code | Deduplicate by Message-ID; block disposable/free domains | Capture Gmail Event | Check Data Exist | (same as above) |
| Check Data Exist | if | Stop workflow when item is empty | Filter + Deduplicate Validation | Arrange Email Data | (same as above) |
| Arrange Email Data | set | Keep only required email fields | Check Data Exist | Find Category | (same as above) |
| Find Category | @n8n/n8n-nodes-langchain.agent | Categorize email to a category (JSON) | Arrange Email Data (+ AI model/parser inputs) | Merge Category | ## AI Email Categorization<br>* The AI Agent node reads the email content and determines its category (for example, Sales, Accounts, etc.).<br>* A predefined list of categories is included in the prompt. If the email matches one of these, the corresponding category name is returned; otherwise, a new category name is generated.<br>* The implementation uses OpenRouter’s LLaMA chat model for classification. |
| OpenRouter Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenRouter | LLM for agent | — | Find Category (ai_languageModel) | (same as above) |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce JSON `{category}` | — | Find Category (ai_outputParser) | (same as above) |
| OpenRouter Chat Model1 | @n8n/n8n-nodes-langchain.lmChatOpenRouter | Extra LLM node (likely unnecessary wiring) | — | Structured Output Parser (ai_languageModel) | (same as above) |
| Merge Category | set | Merge AI category into email payload | Find Category | Get Channels List | ## Merge Node ( Merge Category)<br>* This node merges the category identified by the AI agent into the main data returned by the “Arrange Email Data” node. |
| Get Channels List | slack | Retrieve all Slack channels | Merge Category | Format Channel Data | ## Slack Channels<br>* The Slack node is used to retrieve the list of Slack channels.<br>* The first Code node formats the channel list data returned by the Slack node.<br>* The second Code node filters the formatted channel data, removes duplicate entries, and checks for a matching channel name based on the category provided by the AI agent node.<br>* If a matching channel is found, it returns found: true. |
| Format Channel Data | code | Normalize channels to name/id; dedupe | Get Channels List | Remove Duplicate Channel + Find Matched Channel | (same as above) |
| Remove Duplicate Channel + Find Matched Channel | code | Find channel matching category | Format Channel Data | Check Channel Exist | (same as above) |
| Check Channel Exist | if | Route based on match found | Remove Duplicate Channel + Find Matched Channel | Join Slack Channel / Create Slack Channel | ## Channel Existence Check<br>* This If node checks whether a matching category was found in the previous node.<br>* If found is true, it proceeds with the Existing Channel Flow.<br>* Otherwise, it triggers the New Channel Creation Flow. |
| Join Slack Channel | slack | Join bot to existing channel | Check Channel Exist (true) | Post Message in Channel | ## Existing Channel Flow<br>* The Join Slack Channel node is used to join the app to the matched channel if it has not already joined.<br>* The next Slack node posts the email details to the matched channel. The message body is structured using Block Kit (blocks) to support reply-via-button email functionality, which can be implemented through another workflow.<br>* In message body pass email thread id in button block so ii will used in reply functionality |
| Post Message in Channel | slack | Post Block Kit email summary to matched channel | Join Slack Channel | — | (same as above) |
| Create Slack Channel | slack | Create Slack channel for category | Check Channel Exist (false) | Invite a user to a channel | ## New Channel Creation Flow<br>* It first creates a Slack channel using the category name identified by the AI agent. If the category exists in the predefined category list, that name is used; otherwise, a new channel is created using the AI-generated category name.<br>* Next, it invites the current user to the newly created channel so they can send and view messages.<br>- Finally, a Slack node posts the email details to the channel using Block Kit (blocks) format, including a reply button, whose functionality will be implemented through a separate workflow trigger. |
| Invite a user to a channel | slack | Invite a fixed user ID to channel | Create Slack Channel | Post Message in Created Channel | (same as above) |
| Post Message in Created Channel | slack | Post Block Kit email summary to newly created channel | Invite a user to a channel | — | (same as above) |
| Sticky Note | stickyNote | Comment block | — | — | ##  Gmail Intake & Data Preparation (content as above) |
| Sticky Note1 | stickyNote | Comment block | — | — | ## AI Email Categorization (content as above) |
| Sticky Note2 | stickyNote | Comment block | — | — | ## Merge Node ( Merge Category) (content as above) |
| Sticky Note3 | stickyNote | Comment block | — | — | ## Slack Channels (content as above) |
| Sticky Note4 | stickyNote | Comment block | — | — | ## Channel Existence Check (content as above) |
| Sticky Note5 | stickyNote | Comment block | — | — | ## Existing Channel Flow (content as above) |
| Sticky Note6 | stickyNote | Comment block | — | — | ## New Channel Creation Flow (content as above) |
| Sticky Note7 | stickyNote | Comment block | — | — | ## Prerequisite and Setup Guide (content preserved in section 5) |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name it: **Gmail-to-Slack Email Automation**
- Ensure workflow setting execution order is the default used here (`executionOrder: v1`).

2) **Add Gmail Trigger**
- Node: **Gmail Trigger**
- Credentials: connect Gmail via **OAuth2**
- Configure:
  - Read status: **Unread**
  - Include drafts: **off**
  - Include spam/trash: **off**
  - Download attachments: **on**
  - Polling: **Every minute**
- Output should include headers and message metadata.

3) **Add Code node: “Filter + Deduplicate Validation”**
- Mode: **Run Once for Each Item**
- Paste logic equivalent to:
  - Persist processed Message-IDs in `workflow static data (global)`
  - If duplicate => return empty object
  - Block disposable domains list
  - Block free email provider domains list (optional but enabled here)
- Connect: **Gmail Trigger → Code**

4) **Add IF node: “Check Data Exist”**
- Condition: **Object** → `notEmpty` on `{{$json}}`
- Connect: **Code → IF**
- Use only the **true** output onward.

5) **Add Set node: “Arrange Email Data”**
- Operation: keep only selected fields
- Fields to keep:
  - `id, text, textAsHtml, subject, date, to, from, threadId`
- Connect: **IF (true) → Set**

6) **Add OpenRouter LLM node**
- Node: **OpenRouter Chat Model** (LangChain)
- Credentials: OpenRouter API key
- Model: `meta-llama/llama-3-70b-instruct`

7) **Add Structured Output Parser**
- Node: **Structured Output Parser**
- Enable **Auto-fix**
- Schema example:
  - `{ "category": "sales" }`

8) **Add AI Agent: “Find Category”**
- Node: **AI Agent**
- Prompt type: **Define**
- Prompt: include the predefined categories and rules; inject:
  - `{{$json.subject}}`
  - `{{$json.text}}`
- Require JSON-only output `{ "category": "<category>" }`
- Connect:
  - Main: **Arrange Email Data → Find Category**
  - AI Language Model input: **OpenRouter Chat Model → Find Category**
  - Output Parser input: **Structured Output Parser → Find Category**

9) **Add Set node: “Merge Category”**
- Add fields:
  - `output.category = {{$json.output.category}}`
  - Copy email fields from “Arrange Email Data” using node references (id/text/textAsHtml/subject/date/to/from/threadId)
- Connect: **Find Category → Merge Category**

10) **Add Slack: “Get Channels List”**
- Node: **Slack**
- Credentials: Slack bot token (Slack App)
- Operation: **Channel → Get All**
- Connect: **Merge Category → Get Channels List**

11) **Add Code node: “Format Channel Data”**
- Map channels to `{channel_name, channel_id}` and de-duplicate by id.
- Connect: **Get Channels List → Format Channel Data**

12) **Add Code node: “Remove Duplicate Channel + Find Matched Channel”**
- Read category from `$('Merge Category').first().json.output.category`
- Find matching channel by substring check
- Output `{found, matchedChannelName, matchedChannelId}`
- Connect: **Format Channel Data → Match Code**

13) **Add IF node: “Check Channel Exist”**
- Condition: `{{$json.found}} equals true`
- True branch: existing channel flow
- False branch: new channel flow
- Connect: **Match Code → IF**

### Existing Channel Flow
14) **Slack node: “Join Slack Channel”**
- Operation: **Channel → Join**
- Channel ID: `{{$json.matchedChannelId}}`
- Connect: **Check Channel Exist (true) → Join**

15) **Slack node: “Post Message in Channel”**
- Operation: Post message with **Block Kit**
- Channel ID: `{{$json.id}}` (or safer: use `matchedChannelId`)
- Block Kit message:
  - Include From/Subject/snippet from `$('Merge Category').first()`
  - Add button with `action_id = reply_email` and `value = threadId`
  - Add Gmail link using `id`
- Connect: **Join → Post**

### New Channel Creation Flow
16) **Slack node: “Create Slack Channel”**
- Operation: **Channel → Create**
- Set **channel name** to the category (verify correct field in the Slack node UI; do not accidentally put it in “channelId”).
  - Value: `{{$('Merge Category').first().json.output.category}}`
- Connect: **Check Channel Exist (false) → Create**

17) **Slack node: “Invite a user to a channel”**
- Operation: **Channel → Invite**
- Channel ID: `{{$json.id}}` (from Create)
- User IDs: `["U0A6ULM7CGK"]` (replace with your Slack user ID)
- Connect: **Create → Invite**

18) **Slack node: “Post Message in Created Channel”**
- Operation: Post message with **Block Kit**
- Recommended Channel ID mapping (more robust): reference the Create node’s channel id explicitly.
- Connect: **Invite → Post**

19) **Credentials / scopes checklist**
- **Gmail OAuth2:** read email + metadata (and attachments if enabled).
- **Slack App scopes (typical):**
  - `channels:read`
  - `channels:manage` (create/invite/join depending on Slack settings)
  - `chat:write`
  - Potentially `groups:read`, `groups:write` if private channels involved

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| ## Prerequisite and Setup Guide (full content) | From sticky note “Prerequisite and Setup Guide”: explains end-to-end behavior and high-level setup steps (Gmail OAuth, AI provider config, Slack app scopes, testing & activation). |
| Reply-via-email button design | Both Slack posting nodes include a Block Kit button with `action_id: reply_email` and `value: threadId` to support a separate workflow that would handle Slack interactivity and send replies via Gmail. |


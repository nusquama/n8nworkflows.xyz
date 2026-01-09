Automate follower resource distribution on BlueSky with keyword-triggered DMs

https://n8nworkflows.xyz/workflows/automate-follower-resource-distribution-on-bluesky-with-keyword-triggered-dms-12159


# Automate follower resource distribution on BlueSky with keyword-triggered DMs

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name:** *BlueSky Suite: Auto-DM followers who reply with a keyword*  
**Purpose:** Every 30 minutes, the workflow logs into BlueSky, scans recent *reply* notifications, filters replies that (1) contain a configured keyword and (2) are replies to a specific configured post, deduplicates already-processed replies using persistent workflow memory, then for each qualifying reply: checks whether the author follows you; if yes, opens/gets a chat conversation ID, sends a DM with a configured message, and finally likes the user’s reply.

**Target use cases**
- “Lead magnet” distribution: send a resource link by DM to users who comment a keyword.
- “Follower-gated” fulfillment: only followers receive the DM.
- Automated engagement: liking the reply after sending the DM.

### Logical blocks
1.1 **Scheduling & Configuration Intake** (trigger + config variables)  
1.2 **Authentication** (create BlueSky session, get access JWT)  
1.3 **Notification Fetch & Unpacking** (pull notifications, split into items)  
1.4 **Reply Qualification Filters** (keyword + reply-to-target-post constraints)  
1.5 **Deduplication / Memory Gate** (process only new replies since last run)  
1.6 **Per-Reply Fulfillment Loop** (batch loop, follow check, DM handshake, send DM, like reply)

---

## 2. Block-by-Block Analysis

### 2.1 Scheduling & Configuration Intake

**Overview:** Starts the workflow periodically, and sets all operator-controlled parameters (handle, password, target post URL, keyword, DM text). Extracts the post ID from the target post URL for later matching.

**Nodes involved:**  
- Schedule Trigger  
- Configuration  
- Extract Post ID

#### Node: Schedule Trigger
- **Type / role:** `Schedule Trigger` — workflow entry point on a timer.
- **Configuration:** Runs every **30 minutes**.
- **Connections:** Outputs to **Configuration**.
- **Edge cases / failures:**
  - If the instance is asleep/offline, runs may be delayed or missed.
  - If you expected 15 minutes (sticky note mentions 15), the actual config is 30 minutes.

#### Node: Configuration
- **Type / role:** `Set` — stores user-provided settings centrally.
- **Configuration choices (interpreted):**
  - Defines 5 string fields (intended to be filled by you):
    - `bluesky_handle` (e.g., `steve.bsky.social`)
    - `app_password` (BlueSky app password)
    - `target_post_url` (the post you want to monitor replies to)
    - `trigger_keyword` (the keyword users must include)
    - `dm_message` (message body to DM)
- **Key expressions/variables used elsewhere:**
  - Referenced via `$('Configuration').first().json.<field>`
- **Connections:** Outputs to **Extract Post ID**.
- **Edge cases / failures:**
  - Empty values will cause auth failure, filter mismatches, or expression errors downstream.

#### Node: Extract Post ID
- **Type / role:** `Set` — derives `post_id` from `target_post_url` using regex.
- **Configuration choices (interpreted):**
  - Creates field `post_id` using expression:  
    `$('Configuration').first().json.target_post_url.match(/post\/([a-zA-Z0-9]+)/)[1]`
- **Connections:** Outputs to **BlueSky Auth**.
- **Edge cases / failures:**
  - If `target_post_url` is blank or doesn’t contain `/post/<id>`, `.match(...)` returns `null` and `[1]` throws an expression error.
  - If BlueSky changes URL formats, extraction may break.

---

### 2.2 Authentication

**Overview:** Logs into BlueSky (AT Protocol) and retrieves an `accessJwt` token used to authorize subsequent API calls.

**Nodes involved:**  
- BlueSky Auth

#### Node: BlueSky Auth
- **Type / role:** `HTTP Request` — calls ATProto `createSession` to obtain `accessJwt` and `did`.
- **Configuration choices (interpreted):**
  - **POST** `https://bsky.social/xrpc/com.atproto.server.createSession`
  - Sends JSON body with:
    - `identifier`: `Configuration.bluesky_handle`
    - `password`: `Configuration.app_password`
  - Response is expected to include `accessJwt` and `did`.
- **Key expressions:**
  - `{{$('Configuration').first().json.bluesky_handle}}`
  - `{{ $('Configuration').first().json.app_password }}`
- **Connections:** Outputs to **Get Notifications**.
- **Edge cases / failures:**
  - Invalid handle/app password → 401/403.
  - Rate limiting or temporary outages.
  - Token expiration is handled implicitly by re-auth each run.

---

### 2.3 Notification Fetch & Unpacking

**Overview:** Retrieves recent reply notifications and converts the returned array into per-notification items for filtering and processing.

**Nodes involved:**  
- Get Notifications  
- Split Out

#### Node: Get Notifications
- **Type / role:** `HTTP Request` — fetches notifications.
- **Configuration choices (interpreted):**
  - **GET** `https://bsky.social/xrpc/app.bsky.notification.listNotifications`
  - Query params:
    - `limit=25`
    - `reason=reply`
  - Header:
    - `Authorization: Bearer <accessJwt>`
- **Key expressions:**
  - `=Bearer {{ $('BlueSky Auth').first().json.accessJwt }}`
- **Connections:** Outputs to **Split Out**.
- **Edge cases / failures:**
  - Auth header missing/invalid token → 401.
  - Returns fewer than 25 notifications; may miss older replies if volume is high between runs.
  - API response shape changes (expects `notifications` array downstream).

#### Node: Split Out
- **Type / role:** `Split Out` — expands an array field into individual items.
- **Configuration choices (interpreted):**
  - Splits field `notifications` into multiple items (one per notification).
- **Connections:** Outputs to **Filter Reply contains**.
- **Edge cases / failures:**
  - If `notifications` is absent or not an array, splitting yields 0 items or errors depending on runtime behavior.

---

### 2.4 Reply Qualification Filters

**Overview:** Keeps only notifications that are replies, contain the trigger keyword, and are replies to the configured target post.

**Nodes involved:**  
- Filter Reply contains

#### Node: Filter Reply contains
- **Type / role:** `Filter` — logical gate for eligible notifications.
- **Configuration choices (interpreted):** All conditions must match:
  1. `reason` equals `"reply"`  
  2. `record.text` contains `Configuration.trigger_keyword`  
  3. `record.reply.root.uri` contains extracted `post_id`
- **Key expressions:**
  - `={{ $json.reason }} == "reply"`
  - `={{ $json.record.text }}` contains `={{ $('Configuration').first().json.trigger_keyword }}`
  - `={{ $json.record.reply.root.uri }}` contains `={{ $('Extract Post ID').first().json.post_id }}`
- **Connections:** Outputs to **Filter New Only**.
- **Edge cases / failures:**
  - Case sensitivity: `contains` is case-sensitive here (node option shows `caseSensitive: true`). “linkedin” won’t match “LinkedIn”.
  - Some reply notifications may not include `record.reply.root.uri` (schema differences) → expression may evaluate to empty and fail the intended match.
  - Keyword in other fields (images/alt text) won’t be detected—only `record.text`.

---

### 2.5 Deduplication / Memory Gate

**Overview:** Prevents duplicate processing by remembering the latest processed notification timestamp (`indexedAt`) in persistent workflow static data.

**Nodes involved:**  
- Filter New Only

#### Node: Filter New Only
- **Type / role:** `Code` — deduplication using `global` static data.
- **Configuration choices (interpreted):**
  - Reads: `staticData.lastProcessedTime` (defaults to epoch start).
  - For each item:
    - If `item.json.indexedAt > lastProcessedTime`, keep it.
    - Tracks `maxDate` across kept items.
  - Writes: updates `staticData.lastProcessedTime = maxDate` immediately.
- **Key variables:**
  - `$getWorkflowStaticData('global')`
  - `item.json.indexedAt`
- **Connections:** Outputs to **Loop Over Items**.
- **Edge cases / failures:**
  - **Manual execution behavior:** In many n8n setups, static data may reset during manual testing or be inconsistent depending on how executions are run; duplicates can occur (also called out in sticky note).
  - If notifications arrive out of order, using only `indexedAt` can skip late-arriving older timestamps (rare but possible).
  - If multiple workflow executions run concurrently, updating `lastProcessedTime` early can cause races (one run advances the timestamp and the other skips items).

---

### 2.6 Per-Reply Fulfillment Loop

**Overview:** Processes each qualifying reply one at a time: loads the author profile, checks if they follow you, then (if yes) gets/creates a chat conversation, sends the DM, and likes the reply.

**Nodes involved:**  
- Loop Over Items  
- Check Follow Status  
- Does Follow?  
- Get Convo ID  
- Send DM  
- Like the reply

#### Node: Loop Over Items
- **Type / role:** `Split In Batches` — iterates items sequentially to avoid mixing data between users and to control throughput.
- **Configuration choices (interpreted):**
  - Default batching options (batch size not explicitly set; n8n default behavior applies).
- **Connections:**
  - **Input:** from **Filter New Only**
  - **Output (loop body):** second output goes to **Check Follow Status**
  - **Output (done):** first output is unused (ends branch)
  - **Also receives “continue” input** from:
    - **Does Follow?** false path (skip user, continue loop)
    - **Like the reply** after successful fulfillment (continue loop)
- **Edge cases / failures:**
  - If batch size is large, may hit BlueSky rate limits; if too small, may be slow.
  - Miswiring can cause infinite loops; here it is intentionally wired back to continue.

#### Node: Check Follow Status
- **Type / role:** `HTTP Request` — retrieves the replier’s profile and viewer relationship info.
- **Configuration choices (interpreted):**
  - **GET** `https://bsky.social/xrpc/app.bsky.actor.getProfile`
  - Query: `actor = $json.author.did` (from the notification item being processed)
  - Header: `Authorization: Bearer <accessJwt>`
- **Key expressions:**
  - `={{ $json.author.did }}`
  - `=Bearer {{ $('BlueSky Auth').first().json.accessJwt }}`
- **Connections:** Outputs to **Does Follow?**
- **Edge cases / failures:**
  - If notification lacks `author.did`, request fails.
  - Response may not include `viewer.followedBy` in some contexts (privacy/blocked users), affecting the IF condition.

#### Node: Does Follow?
- **Type / role:** `IF` — gates fulfillment to followers only.
- **Configuration choices (interpreted):**
  - Condition: `$json.viewer.followedBy` is **not empty**.
  - **True branch:** follower → proceed to **Get Convo ID**
  - **False branch:** not follower → go back to **Loop Over Items** to continue with next item
- **Key expressions:**
  - `={{ $json.viewer.followedBy }}`
- **Edge cases / failures:**
  - If `viewer` object is missing, the expression may resolve empty and route to false (skip).
  - “followedBy” semantics can change; verify with current BlueSky profile schema.

#### Node: Get Convo ID
- **Type / role:** `HTTP Request` — chat “handshake” to find/create a conversation with the user.
- **Configuration choices (interpreted):**
  - **GET** `https://api.bsky.chat/xrpc/chat.bsky.convo.getConvoForMembers`
  - Query: `members = <user did>`
  - Header: `Authorization: Bearer <accessJwt>`
- **Key expressions:**
  - `members = {{ $('Check Follow Status').item.json.did }}`
    - Note: This references the `did` returned by *Check Follow Status*, not the notification directly.
- **Connections:** Outputs to **Send DM**
- **Edge cases / failures:**
  - Chat API may require additional scopes/eligibility; may return 401/403 even if main bsky API works.
  - If the chat service domain changes (`api.bsky.chat`), requests fail.
  - If response doesn’t include `$json.convo.id`, the next node fails.

#### Node: Send DM
- **Type / role:** `HTTP Request` — sends the configured DM text to the conversation.
- **Configuration choices (interpreted):**
  - **POST** `https://api.bsky.chat/xrpc/chat.bsky.convo.sendMessage`
  - JSON body includes:
    - `convoId`: `{{$json.convo.id}}` (from Get Convo ID response)
    - `message.text`: `Configuration.dm_message`
  - Header: `Authorization: Bearer <accessJwt>`
- **Connections:** Outputs to **Like the reply**
- **Edge cases / failures:**
  - DM restrictions (recipient settings, blocks) → API error.
  - If `dm_message` is empty, it may send blank content or fail validation.
  - Rate limits if many replies processed at once.

#### Node: Like the reply
- **Type / role:** `HTTP Request` — creates a “like” record for the reply post.
- **Configuration choices (interpreted):**
  - **POST** `https://bsky.social/xrpc/com.atproto.repo.createRecord`
  - Body:
    - `repo`: authenticated user DID (`BlueSky Auth.did`)
    - `collection`: `app.bsky.feed.like`
    - `record.createdAt`: current ISO timestamp
    - `record.subject.uri` and `record.subject.cid`: from the *current loop item* (`Loop Over Items`.item.json.uri/cid)
  - Header: `Authorization: Bearer <accessJwt>`
- **Key expressions:**
  - `repo`: `{{ $('BlueSky Auth').first().json.did }}`
  - `subject.uri`: `{{ $('Loop Over Items').item.json.uri }}`
  - `subject.cid`: `{{ $('Loop Over Items').item.json.cid }}`
- **Connections:** Outputs back to **Loop Over Items** to continue.
- **Edge cases / failures:**
  - If `uri`/`cid` are missing on the notification item, like creation fails.
  - Duplicate likes may return an error depending on API behavior.
  - Time skew is usually tolerated but could matter in strict validations.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | scheduleTrigger | Periodic entry point (30 min) | — | Configuration | # Input Processing |
| Configuration | set | Central runtime configuration values | Schedule Trigger | Extract Post ID | # 🚀 Lead Magnet / Auto-DM Bot - How To Use \n**Goal:** Automatically DM a link/resource to anyone who replies to your posts with a specific keyword (e.g., "LinkedIn").\n\n**Step 1:** Campaign Setup Open the "Configuration" node (first green node) and set:\n\n- Handle & Password: Your BlueSky credentials.\n- Trigger Keyword: The word users must type (e.g., Template).\n- DM Message: The message they will receive (e.g., "Hey! Here is the link: ...").\n\n\n**Step 2:** The "Growth Hack" Logic This workflow checks 3 things before sending a message:\n\n- Did they use the Keyword?\n- Is this a New reply? (Prevents duplicate DMs).\n- Do they Follow you? (Incentivizes followers).\n\n\n**Step 3:** Activate Turn it to "Active". It runs every 15 minutes, scanning your notifications for new leads.\n\n[**Sample Google Sheet**](https://docs.google.com/spreadsheets/d/1Mg04gK1K5DBtJHrWw3ePRFc_JjkxwAp0deGjapVl2q0/edit?usp=sharing)\n\n\n\n## ⚠️ READ BEFORE TESTING\n\n**In Manual Testing Mode:**\nn8n wipes the memory (`staticData`) every time you click "Execute". The bot will **forget** previous runs and may send duplicate DMs to the same user during manual tests.\n\n**To Test Deduplication Properly:**\n1. **Activate** the workflow (Toggle "Active" in top right).\n2. Reply to your post on BlueSky using the keyword.\n3. Wait for the workflow to run automatically (every 15 min).\n4. Check "Executions" to confirm it only processed the reply **once**.\n---\n### 1- START HERE \nEnter your \n- BlueSky Handle (e.g., steve.bsky.social)\n- App Password. \nThis sets up the authentication for the whole workflow.\n- Trigger Keyword: The word users must type (e.g., Linkedin).\n- DM text: Message to send followers when they reply with the "Trigger Keyword"\n# Input Processing |
| Extract Post ID | set | Extracts post_id from target_post_url | Configuration | BlueSky Auth | # Input Processing |
| BlueSky Auth | httpRequest | Authenticates and returns accessJwt/did | Extract Post ID | Get Notifications | ### 2- Get access token\nLogin to BlueSky using handle and app password to get the access token. This will be used in all future BlueSky api calls.\n# Input Processing |
| Get Notifications | httpRequest | Fetches last 25 reply notifications | BlueSky Auth | Split Out | # Input Processing |
| Split Out | splitOut | Explodes notifications array into items | Get Notifications | Filter Reply contains | ### 4- The Unpacker \nThe BlueSky API sends us one big list containing 25 notifications. This node "explodes" that list into 25 separate items so we can filter and process each notification individually.\n# Input Processing |
| Filter Reply contains | filter | Filters by keyword + reply-to-target-post | Split Out | Filter New Only | ### 5- The Keyword Filter \nFilters the stream to only allow:\n- Replies (ignores likes/reposts).\n- Text that contains your Trigger keyword (from the Config node).\n# Input Processing |
| Filter New Only | code | Deduplication via staticData timestamp | Filter Reply contains | Loop Over Items | ### 6- The Memory Gatekeeper \n**⚠️ System Node: Handles deduplication. No configuration needed here.**\n\nThis script remembers the timestamp of the last processed reply. It ensures that even if the workflow runs 100 times, it never sends a duplicate DM to the same user for the same reply.\n# Input Processing |
| Loop Over Items | splitInBatches | Sequential per-reply processor + loop control | Filter New Only; Does Follow? (false); Like the reply | Check Follow Status (loop body) | ### 7- The Processor \nTakes the "Winning Replies" (those that passed the keyword and duplicate checks) and handles them one by one. \n\nThis ensures that the variables for User A don't get mixed up with User B.\n# Fulfillment Logic |
| Check Follow Status | httpRequest | Loads replier profile/viewer relationship | Loop Over Items | Does Follow? | ### 8- The Growth Hook \nWe check if the user follows you.\n\nIf YES: They get the DM.\n\nIf NO: We skip them. (This encourages users to follow you to get the resource).\n# Fulfillment Logic |
| Does Follow? | if | Follower gate | Check Follow Status | Get Convo ID (true); Loop Over Items (false) | ### 9- The Growth Gate \nThis checks the data from the previous node ("Check Follow Status").\n\nTrue Path: The user follows you → Send them the DM.\n\nFalse Path: They don't follow → Stop. (This effectively "locks" your lead magnet behind a follow).\n# Fulfillment Logic |
| Get Convo ID | httpRequest | Gets/creates chat convo with member DID | Does Follow? (true) | Send DM | ### 10- The Handshake \nYou cannot send a DM directly to a User ID. \n\nYou must first ask the Chat API to find (or create) a Conversation ID between you and that user. \n\nThis node performs that "handshake."\n# Fulfillment Logic |
| Send DM | httpRequest | Sends DM with configured message | Get Convo ID | Like the reply | ### 11- The Delivery \nUses the Conversation ID we just retrieved to send the actual text message. \n\n**Note:** The message text is pulled dynamically from your Configuration node at the start.\n# Fulfillment Logic |
| Like the reply | httpRequest | Likes the reply as confirmation/engagement | Send DM | Loop Over Items | ### 12- Like the reply\nThe "Delight" Factor Before finishing, we "Like" the user's reply. \n\nThis gives them a notification "Creator liked your reply", confirming you saw them, even if the DM takes a moment to arrive.\n# Fulfillment Logic |
| Sticky Note | stickyNote | Documentation | — | — |  |
| Sticky Note1 | stickyNote | Documentation | — | — |  |
| Sticky Note2 | stickyNote | Documentation | — | — |  |
| Sticky Note3 | stickyNote | Documentation | — | — |  |
| Sticky Note4 | stickyNote | Documentation | — | — |  |
| Sticky Note5 | stickyNote | Documentation | — | — |  |
| Sticky Note6 | stickyNote | Documentation | — | — |  |
| Sticky Note7 | stickyNote | Documentation | — | — |  |
| Sticky Note8 | stickyNote | Documentation | — | — |  |
| Sticky Note9 | stickyNote | Documentation | — | — |  |
| Sticky Note10 | stickyNote | Documentation | — | — |  |
| Sticky Note11 | stickyNote | Documentation | — | — |  |
| Sticky Note12 | stickyNote | Documentation | — | — |  |
| Sticky Note13 | stickyNote | Documentation | — | — |  |
| Sticky Note14 | stickyNote | Documentation | — | — |  |

> Note: Sticky notes are included as nodes in the workflow JSON; they do not execute.

---

## 4. Reproducing the Workflow from Scratch

1) **Create “Schedule Trigger” (Schedule Trigger node)**
   - Set interval: every **30 minutes**.

2) **Create “Configuration” (Set node)**
   - Add string fields:
     - `bluesky_handle`
     - `app_password`
     - `target_post_url`
     - `trigger_keyword`
     - `dm_message`
   - Fill them with your values (or leave blank and fill later).
   - Connect: **Schedule Trigger → Configuration**.

3) **Create “Extract Post ID” (Set node)**
   - Add field `post_id` (string) with expression:  
     `$('Configuration').first().json.target_post_url.match(/post\/([a-zA-Z0-9]+)/)[1]`
   - Connect: **Configuration → Extract Post ID**.

4) **Create “BlueSky Auth” (HTTP Request node)**
   - Method: **POST**
   - URL: `https://bsky.social/xrpc/com.atproto.server.createSession`
   - Body content type: JSON
   - JSON body:
     - `identifier`: expression `{{$('Configuration').first().json.bluesky_handle}}`
     - `password`: expression `{{ $('Configuration').first().json.app_password }}`
   - Connect: **Extract Post ID → BlueSky Auth**.

5) **Create “Get Notifications” (HTTP Request node)**
   - Method: **GET**
   - URL: `https://bsky.social/xrpc/app.bsky.notification.listNotifications`
   - Query params:
     - `limit` = `25`
     - `reason` = `reply`
   - Headers:
     - `Authorization` = `Bearer {{ $('BlueSky Auth').first().json.accessJwt }}`
   - Connect: **BlueSky Auth → Get Notifications**.

6) **Create “Split Out” (Split Out node)**
   - Field to split out: `notifications`
   - Connect: **Get Notifications → Split Out**.

7) **Create “Filter Reply contains” (Filter node)**
   - Add conditions (AND):
     1. String equals: left `{{$json.reason}}` right `reply`
     2. String contains: left `{{$json.record.text}}` right `{{$('Configuration').first().json.trigger_keyword}}`
     3. String contains: left `{{$json.record.reply.root.uri}}` right `{{$('Extract Post ID').first().json.post_id}}`
   - Ensure case sensitivity matches your intent (current workflow is case-sensitive).
   - Connect: **Split Out → Filter Reply contains**.

8) **Create “Filter New Only” (Code node)**
   - Paste the dedupe script (as authored) that:
     - reads `staticData.lastProcessedTime`
     - filters by `item.json.indexedAt`
     - updates `staticData.lastProcessedTime`
   - Connect: **Filter Reply contains → Filter New Only**.

9) **Create “Loop Over Items” (Split In Batches node)**
   - Keep default options (or set batch size explicitly if desired).
   - Connect: **Filter New Only → Loop Over Items**.
   - Use the **loop/body output** (the second output in this workflow) to continue processing.

10) **Create “Check Follow Status” (HTTP Request node)**
   - Method: **GET**
   - URL: `https://bsky.social/xrpc/app.bsky.actor.getProfile`
   - Query params:
     - `actor` = `{{$json.author.did}}`
   - Header:
     - `Authorization` = `Bearer {{ $('BlueSky Auth').first().json.accessJwt }}`
   - Connect: **Loop Over Items (loop output) → Check Follow Status**.

11) **Create “Does Follow?” (IF node)**
   - Condition: string “not empty”
     - value: `{{$json.viewer.followedBy}}`
   - Connect: **Check Follow Status → Does Follow?**
   - False branch connection: **Does Follow? (false) → Loop Over Items** (to continue next item).

12) **Create “Get Convo ID” (HTTP Request node)**
   - Method: **GET**
   - URL: `https://api.bsky.chat/xrpc/chat.bsky.convo.getConvoForMembers`
   - Query params:
     - `members` = `{{ $('Check Follow Status').item.json.did }}`
   - Header:
     - `Authorization` = `Bearer {{ $('BlueSky Auth').first().json.accessJwt }}`
   - Connect: **Does Follow? (true) → Get Convo ID**.

13) **Create “Send DM” (HTTP Request node)**
   - Method: **POST**
   - URL: `https://api.bsky.chat/xrpc/chat.bsky.convo.sendMessage`
   - JSON body:
     - `convoId` = `{{$json.convo.id}}`
     - `message.text` = `{{$('Configuration').first().json.dm_message}}`
   - Header:
     - `Authorization` = `Bearer {{ $('BlueSky Auth').first().json.accessJwt }}`
   - Connect: **Get Convo ID → Send DM**.

14) **Create “Like the reply” (HTTP Request node)**
   - Method: **POST**
   - URL: `https://bsky.social/xrpc/com.atproto.repo.createRecord`
   - JSON body:
     - `repo` = `{{ $('BlueSky Auth').first().json.did }}`
     - `collection` = `app.bsky.feed.like`
     - `record`:
       - `$type` = `app.bsky.feed.like`
       - `createdAt` = `{{ new Date().toISOString() }}`
       - `subject.uri` = `{{ $('Loop Over Items').item.json.uri }}`
       - `subject.cid` = `{{ $('Loop Over Items').item.json.cid }}`
   - Header:
     - `Authorization` = `Bearer {{ $('BlueSky Auth').first().json.accessJwt }}`
   - Connect: **Send DM → Like the reply**.
   - Connect: **Like the reply → Loop Over Items** (continue loop).

15) **Credentials**
   - No n8n credential objects are used; authentication is handled by HTTP requests using `Configuration` values.
   - Ensure your BlueSky **App Password** is valid and permitted for these endpoints.
   - If your environment restricts outbound calls, allow:
     - `bsky.social` and `api.bsky.chat`

16) **Activate**
   - Toggle workflow to **Active** to benefit from persistent deduplication behavior.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Sample Google Sheet link is present in the notes, but this workflow JSON contains no Google Sheets nodes; the note appears to be from a broader suite/template. | https://docs.google.com/spreadsheets/d/1Mg04gK1K5DBtJHrWw3ePRFc_JjkxwAp0deGjapVl2q0/edit?usp=sharing |
| The notes claim “runs every 15 minutes”, but the Schedule Trigger is configured for every 30 minutes. Align one of them depending on intent. | Sticky note content vs Schedule Trigger configuration |
| Manual executions may not preserve `staticData` consistently; duplicates can occur during testing. Proper dedupe validation requires running the workflow as Active on schedule. | Sticky note “READ BEFORE TESTING” |
| “Sticky Note4” mentions Google Sheets “Posted” rows, but there is no Google Sheets integration in this workflow—safe to ignore or remove. | Internal comment mismatch with actual nodes |


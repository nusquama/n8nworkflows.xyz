Route AI-powered customer support via email and Slack with PostgreSQL and Google Sheets

https://n8nworkflows.xyz/workflows/route-ai-powered-customer-support-via-email-and-slack-with-postgresql-and-google-sheets-12891


# Route AI-powered customer support via email and Slack with PostgreSQL and Google Sheets

## 1. Workflow Overview

**Title:** Route AI-powered customer support via email and Slack with PostgreSQL and Google Sheets

**Purpose & use cases:**  
This workflow receives customer support questions via an HTTP webhook, analyzes sentiment/urgency, optionally alerts Slack for urgent cases, builds context from a PostgreSQL conversation history plus a (mock) knowledge base search, generates an AI-style response with confidence scoring, routes either to an automated email reply or to a Slack escalation, logs the interaction to Google Sheets and PostgreSQL, and finally returns a structured JSON response to the webhook caller.

### 1.1 Logical Blocks (by dependencies)
1. **Intake & Normalization**: Webhook Trigger → Extract & Enrich Data  
2. **Sentiment / Urgency Analysis**: Sentiment & Context Analysis → Urgency Check  
3. **Urgent Alert Path + Merge**: (Urgent) Send Urgent Alert → Merge After Urgency; (Non-urgent) → Merge After Urgency  
4. **Context Building & Response Generation**: Fetch Conversation History → Search Knowledge Base → Build AI Context → AI Response Generator → Process & Analyze Response  
5. **Smart Routing & Delivery**: Routing Decision → (Escalate) Slack Escalation OR (Auto) Send Auto Response → Merge After Routing  
6. **Logging & Final API Response**: Log to Analytics Dashboard → Save to Database → Send API Response  
7. **Documentation notes (sticky notes)**: Multiple sticky note nodes describing sections and setup.

---

## 2. Block-by-Block Analysis

### Block 1 — Intake & Normalization
**Overview:** Receives an inbound support request via webhook and normalizes the payload into consistent fields (IDs, timestamps, defaults).

**Nodes involved:**  
- Webhook Trigger  
- Extract & Enrich Data  

#### Node: Webhook Trigger
- **Type / role:** `Webhook` trigger; entry point for the workflow.
- **Configuration choices:**
  - **Method:** POST
  - **Path:** `/customer-support`
  - **Response mode:** “Respond with Respond to Webhook node” (responseNode)
- **Key variables / payload expectations:** expects JSON body with keys like `email`, `name`, `question`, optional `conversationId`, optional `language`, optional `priority`.
- **Connections:**
  - **Output →** Extract & Enrich Data
- **Version-specific:** typeVersion `2` (standard modern webhook node behavior).
- **Failure modes / edge cases:**
  - Missing `body` or missing `body.email` / `body.question` will break downstream expressions (e.g., `.split('@')[0]`).
  - If caller sends non-JSON or wrong content-type, `$json.body` may not exist.

#### Node: Extract & Enrich Data
- **Type / role:** `Set` node; maps/creates normalized fields.
- **Configuration choices (interpreted):**
  - Creates:
    - `questionId` generated from ISO timestamp + email local-part + random suffix
    - `userQuestion`, `userEmail`, `userName` (default “Customer”)
    - `timestamp` (`$now.toISO()`)
    - `conversationId` (provided or `new-<millis>`)
    - `userLanguage` (provided or `en`)
    - `priority` (provided or `normal`)
- **Key expressions:**
  - `questionId`: `{{ $now.toISO() }}-{{ $json.body.email.split('@')[0] }}-{{ Math.random().toString(36).substr(2, 9) }}`
  - `conversationId`: `{{ $json.body.conversationId || 'new-' + $now.toMillis() }}`
- **Connections:**
  - **Input ←** Webhook Trigger
  - **Output →** Sentiment & Context Analysis
- **Version-specific:** typeVersion `3.3` (Set node with “assignments” model).
- **Failure modes / edge cases:**
  - `email.split('@')[0]` throws if email missing or not a string.
  - Using `$now` is fine; but `Math.random()` makes IDs non-deterministic (harder to dedupe).

---

### Block 2 — Sentiment / Urgency Analysis
**Overview:** Performs a lightweight (mock) sentiment analysis, urgency detection, keyword extraction, and language detection (Japanese vs English).

**Nodes involved:**  
- Sentiment & Context Analysis  
- Urgency Check  

#### Node: Sentiment & Context Analysis
- **Type / role:** `Code` node; custom JS transformation.
- **Configuration choices:**
  - Keyword-based sentiment scoring:
    - negative/positive keyword lists adjust `emotionScore`
    - urgent keyword list toggles `urgency = high`
  - Sets `requiresImmediateAttention` if very negative or urgency high
  - Keyword extraction: tokenization, basic filtering, first 5 tokens
  - Language detection: checks for Kanji/Hiragana → `ja` else `en`
- **Key outputs:**
  - `sentiment`, `emotionScore`, `urgency`, `keywords`, `detectedLanguage`, `requiresImmediateAttention`
- **Connections:**
  - **Input ←** Extract & Enrich Data
  - **Output →** Urgency Check
- **Version-specific:** typeVersion `2` (Code node).
- **Failure modes / edge cases:**
  - If `userQuestion` is undefined/not string, `.toLowerCase()` fails.
  - This is heuristic; false positives/negatives can affect escalation routing.

#### Node: Urgency Check
- **Type / role:** `IF` node; branches urgent vs normal.
- **Configuration choices:**
  - Condition: `{{ $json.requiresImmediateAttention }}` is `true`
- **Connections:**
  - **Input ←** Sentiment & Context Analysis
  - **True output →** Send Urgent Alert
  - **False output →** Merge After Urgency (into input 2, index 1)
- **Version-specific:** typeVersion `2`.
- **Failure modes / edge cases:**
  - If `requiresImmediateAttention` missing/not boolean, strict validation may route unexpectedly.

---

### Block 3 — Urgent Alert Path + Merge
**Overview:** If urgent, send a Slack webhook alert immediately; regardless of urgency, merge back into a single stream to continue processing.

**Nodes involved:**  
- Send Urgent Alert  
- Merge After Urgency  

#### Node: Send Urgent Alert
- **Type / role:** `HTTP Request` node; posts Slack webhook message for urgent cases.
- **Configuration choices:**
  - POST to `https://hooks.slack.com/services/YOUR_URGENT_WEBHOOK` (placeholder)
  - JSON body with Slack Block Kit layout, includes question, user info, keywords, timestamp
- **Key expressions:**
  - Uses templating in JSON body, e.g. `{{ $json.userName }}`, `{{ $json.questionId }}`
- **Connections:**
  - **Input ←** Urgency Check (true branch)
  - **Output →** Merge After Urgency (input 1, index 0)
- **Version-specific:** typeVersion `4.2`.
- **Failure modes / edge cases:**
  - Invalid Slack webhook URL returns 4xx.
  - Slack rate-limits or payload formatting errors (invalid blocks).
  - Note: JSON includes emoji characters; Slack generally supports them.

#### Node: Merge After Urgency
- **Type / role:** `Merge` node; joins urgent/non-urgent paths.
- **Configuration choices:**
  - Default merge behavior (no explicit mode shown). With two inputs connected, it will proceed when it receives data (behavior depends on n8n merge mode defaults).
- **Connections:**
  - **Input 1 ←** Send Urgent Alert
  - **Input 2 ←** Urgency Check (false branch)
  - **Output →** Fetch Conversation History
- **Version-specific:** typeVersion `3`.
- **Failure modes / edge cases:**
  - Merge behavior can be surprising if both branches could run; here only one branch runs, so it effectively “passes through” the active branch.

---

### Block 4 — Context Building & Response Generation
**Overview:** Retrieves recent conversation records from PostgreSQL, performs a mock knowledge base search, composes an AI prompt/context, generates a mock AI response including metadata, then parses it to compute routing decisions.

**Nodes involved:**  
- Fetch Conversation History  
- Search Knowledge Base  
- Build AI Context  
- AI Response Generator  
- Process & Analyze Response  

#### Node: Fetch Conversation History
- **Type / role:** `Postgres` node; reads recent history.
- **Configuration choices:**
  - Executes SQL selecting `conversation_history, created_at` from `support_conversations`
  - Filters by `conversation_id` OR `user_email`
  - Orders newest first, limit 5
  - Note in node: “Disable if table doesn’t exist”
- **Key expressions:**
  - SQL is built via string interpolation using `{{ $json.conversationId }}` and `{{ $json.userEmail }}`
- **Connections:**
  - **Input ←** Merge After Urgency
  - **Output →** Search Knowledge Base
- **Version-specific:** typeVersion `2.4`.
- **Failure modes / edge cases:**
  - Table/column missing: query fails.
  - **SQL injection risk:** direct string interpolation of user-controlled values (email, conversationId). Use parameterized queries in production.
  - Credential/network errors to DB.

#### Node: Search Knowledge Base
- **Type / role:** `Code` node; mock KB retrieval (simulating vector search).
- **Configuration choices:**
  - Chooses mock results depending on `detectedLanguage` (`ja` vs `en`)
  - Adds `searchResults` array to the JSON
- **Connections:**
  - **Input ←** Fetch Conversation History
  - **Output →** Build AI Context
- **Failure modes / edge cases:**
  - If `detectedLanguage` missing, defaults to the English branch due to comparison.
  - In real vector search integration, expect timeouts and empty results.

#### Node: Build AI Context
- **Type / role:** `Code` node; builds prompt and contextual strings.
- **Configuration choices:**
  - Pulls conversation history using: `$('Fetch Conversation History').all()`
  - Formats:
    - `historyContext` (if any history items have `conversation_history`)
    - `knowledgeContext` from `searchResults` (includes relevance and source)
  - Determines tone guidance based on sentiment
  - Builds `aiPrompt` including instructions to output metadata lines:
    - `CONFIDENCE_SCORE: [0-100]`
    - `REQUIRES_FOLLOWUP: [true/false]`
    - `SUGGESTED_CATEGORY: [category name]`
- **Connections:**
  - **Input ←** Search Knowledge Base
  - **Output →** AI Response Generator
- **Failure modes / edge cases:**
  - If Postgres returns no items, historyContext remains empty (handled).
  - If `searchResults` not an array, it falls back to “No relevant information found.”
  - The cross-node reference `$('Fetch Conversation History')` requires that node to have executed in the same run (it does in this design).

#### Node: AI Response Generator
- **Type / role:** `Code` node; mock AI response generator.
- **Configuration choices:**
  - Creates a response in Japanese or English.
  - Computes confidence score from average `searchResults[].score`, default 0.3.
  - Sets `requiresFollowup` based on confidence, sentiment, and whether search results exist.
  - Categorizes by keyword matching (password/payment/bug/refund, incl. Japanese variants).
  - Appends metadata lines required by downstream parsing.
- **Connections:**
  - **Input ←** Build AI Context
  - **Output →** Process & Analyze Response
- **Failure modes / edge cases:**
  - If `searchResults` missing/empty, confidence becomes low; triggers escalation more often.
  - In production, replace with LLM node; ensure the output format is stable or change the parser.

#### Node: Process & Analyze Response
- **Type / role:** `Code` node; parses metadata, cleans response, computes escalation/priority.
- **Configuration choices:**
  - Regex parsing:
    - `CONFIDENCE_SCORE:\s*(\d+)`
    - `REQUIRES_FOLLOWUP:\s*(true|false)`
    - `SUGGESTED_CATEGORY:\s*(.+)`
  - Removes metadata from the email/slack response (`cleanResponse`)
  - `needsEscalation` if:
    - confidence < 70, or requiresFollowup, or very negative, or urgency high, or “escalate” in text, or no KB results
  - Computes `priorityScore` and maps to `priority` (`critical/high/medium/low`)
  - Adds `estimatedResponseTime`
  - Outputs a flattened “case record” used by routing + logging
- **Connections:**
  - **Input ←** AI Response Generator
  - **Output →** Routing Decision
- **Failure modes / edge cases:**
  - If AI output deviates from metadata format, regex may fail; confidence defaults to 0 which forces escalation.
  - Metadata stripping could remove unintended text if the patterns occur naturally.

---

### Block 5 — Smart Routing & Delivery
**Overview:** Routes escalations to Slack; otherwise sends an automated email. Both paths then merge to a single logging chain.

**Nodes involved:**  
- Routing Decision  
- Slack Escalation  
- Send Auto Response  
- Merge After Routing  

#### Node: Routing Decision
- **Type / role:** `IF` node; chooses escalation vs auto-respond.
- **Configuration choices:**
  - Condition: `{{ $json.needsEscalation }}` is `true`
- **Connections:**
  - **True output →** Slack Escalation
  - **False output →** Send Auto Response
- **Failure modes / edge cases:**
  - If `needsEscalation` missing, strict validation may treat it as false or error depending on runtime; here it’s always set by previous node.

#### Node: Slack Escalation
- **Type / role:** `HTTP Request`; posts detailed escalation to Slack.
- **Configuration choices:**
  - POST to `https://hooks.slack.com/services/YOUR_ESCALATION_WEBHOOK` (placeholder)
  - Rich Block Kit message includes:
    - priority indicator, questionId, user info, category, sentiment, confidence
    - question and truncated suggested response (first 500 chars)
    - action links to dashboards
- **Connections:**
  - **Input ←** Routing Decision (true branch)
  - **Output →** Merge After Routing (input 1)
- **Failure modes / edge cases:**
  - Slack webhook errors/limits.
  - Block Kit JSON must remain valid; long fields may be truncated by Slack.

#### Node: Send Auto Response
- **Type / role:** `Email Send`; sends customer-facing email.
- **Configuration choices:**
  - **To:** `{{ $json.userEmail }}`
  - **From:** `support@example.com` (placeholder)
  - **Subject:** language-aware, includes category  
    `ja`: “お問い合わせありがとうございます - <category>”  
    `en`: “Thank you for your inquiry - <category>”
  - **Important:** This node as provided does **not** set the email body/text parameter. It will likely send a blank email unless n8n defaults are configured elsewhere.
- **Connections:**
  - **Input ←** Routing Decision (false branch)
  - **Output →** Merge After Routing (input 2)
- **Version-specific:** typeVersion `2.1`.
- **Failure modes / edge cases:**
  - Missing SMTP/Gmail credentials, invalid from address, provider restrictions.
  - If body is not configured, customer receives empty message (functional defect to fix).

#### Node: Merge After Routing
- **Type / role:** `Merge`; joins escalation vs auto-response paths for unified logging.
- **Connections:**
  - **Input 1 ←** Slack Escalation
  - **Input 2 ←** Send Auto Response
  - **Output →** Log to Analytics Dashboard
- **Failure modes / edge cases:**
  - Same merge considerations as earlier: only one branch should execute.

---

### Block 6 — Logging & Final API Response
**Overview:** Logs the case to Google Sheets and PostgreSQL, then returns a structured JSON payload to the webhook caller.

**Nodes involved:**  
- Log to Analytics Dashboard  
- Save to Database  
- Send API Response  

#### Node: Log to Analytics Dashboard
- **Type / role:** `Google Sheets`; append or update analytics row.
- **Configuration choices:**
  - Operation: `appendOrUpdate`
  - Document: `YOUR_SPREADSHEET_ID` (placeholder)
  - Sheet: `gid=0` (first sheet, selected by list mode)
  - Matching column: `questionId`
  - Writes columns including: status, urgency, category, keywords, language, priority, question, userName, sentiment, timestamp, userEmail, aiResponse, confidenceScore, etc.
- **Connections:**
  - **Input ←** Merge After Routing
  - **Output →** Save to Database
- **Version-specific:** typeVersion `4.4`.
- **Failure modes / edge cases:**
  - Auth failure (Google OAuth), missing spreadsheet permissions.
  - Column mismatch if sheet headers don’t align to expected names.
  - `appendOrUpdate` relies on `questionId` uniqueness/consistency.

#### Node: Save to Database
- **Type / role:** `Postgres`; persists the interaction.
- **Configuration choices:**
  - INSERT into `support_conversations` with many fields (conversation_id, question_id, user details, question, ai_response, etc.)
  - `ON CONFLICT (question_id) DO UPDATE` updates ai_response, confidence_score, status, updated_at.
- **Connections:**
  - **Input ←** Log to Analytics Dashboard
  - **Output →** Send API Response
- **Version-specific:** typeVersion `2.4`.
- **Failure modes / edge cases:**
  - Table schema must match referenced columns and have a unique constraint on `question_id`.
  - **SQL injection risk:** direct string interpolation of question/response/email/name into SQL. This is high risk because `userQuestion` and `aiResponse` can contain quotes/newlines. Use parameterized queries or escape properly.
  - Large response text may exceed column limits.

#### Node: Send API Response
- **Type / role:** `Respond to Webhook`; final HTTP response.
- **Configuration choices:**
  - Respond with JSON object including:
    - `success`, `questionId`, `conversationId`, `status`, `priority`
    - localized `message` based on status + language
    - `response` (clean AI response)
    - `metadata`: confidenceScore, category, sentiment, language, requiresFollowup, estimatedResponseTime
- **Connections:**
  - **Input ←** Save to Database
  - **Output:** ends workflow / returns HTTP response to the original webhook request
- **Version-specific:** typeVersion `1.1`.
- **Failure modes / edge cases:**
  - If earlier nodes fail and execution stops, webhook caller may not receive response (depending on n8n error settings).
  - Response includes AI text; ensure it’s safe/expected for clients.

---

### Block 7 — Sticky Notes / In-Canvas Documentation
**Overview:** These nodes are documentation-only and do not affect execution.

**Nodes involved:**  
- Sticky Note  
- Sticky Note1  
- Sticky Note2  
- Sticky Note3  
- Sticky Note4  
- Sticky Note5  

**Common considerations:**
- No inputs/outputs (not connected).
- Contain setup instructions and section descriptions that are important for reproduction.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Canvas documentation (overall description & setup) |  |  | ## Automate intelligent customer support responses with AI and Slack… (includes setup steps and features; placeholders: `YOUR_URGENT_WEBHOOK`, `YOUR_ESCALATION_WEBHOOK`, `YOUR_SPREADSHEET_ID`) |
| Sticky Note1 | Sticky Note | Canvas documentation (block label) |  |  | ## Intake & Analysis — Receives customer requests, extracts data, and analyzes sentiment/urgency… |
| Sticky Note2 | Sticky Note | Canvas documentation (block label) |  |  | ## Urgency Routing — Sends immediate Slack alerts for urgent cases… |
| Sticky Note3 | Sticky Note | Canvas documentation (block label) |  |  | ## Context Building & AI Response — Fetches conversation history, searches knowledge base… |
| Sticky Note4 | Sticky Note | Canvas documentation (block label) |  |  | ## Smart Routing & Delivery — Decides: auto-respond via email OR escalate to Slack… |
| Sticky Note5 | Sticky Note | Canvas documentation (block label) |  |  | ## Logging & Response — Logs to Google Sheets and PostgreSQL… |
| Webhook Trigger | Webhook | Receives inbound support request | — | Extract & Enrich Data |  |
| Extract & Enrich Data | Set | Normalizes input + generates IDs | Webhook Trigger | Sentiment & Context Analysis |  |
| Sentiment & Context Analysis | Code | Sentiment/urgency detection + language detection | Extract & Enrich Data | Urgency Check |  |
| Urgency Check | IF | Branches urgent vs normal | Sentiment & Context Analysis | Send Urgent Alert; Merge After Urgency |  |
| Send Urgent Alert | HTTP Request | Slack urgent alert | Urgency Check (true) | Merge After Urgency |  |
| Merge After Urgency | Merge | Re-joins urgent/non-urgent flow | Send Urgent Alert; Urgency Check (false) | Fetch Conversation History |  |
| Fetch Conversation History | Postgres | Load recent conversation history | Merge After Urgency | Search Knowledge Base |  |
| Search Knowledge Base | Code | Mock KB search (vector-search placeholder) | Fetch Conversation History | Build AI Context |  |
| Build AI Context | Code | Build prompt/context from KB + DB history | Search Knowledge Base | AI Response Generator |  |
| AI Response Generator | Code | Mock AI response + metadata | Build AI Context | Process & Analyze Response |  |
| Process & Analyze Response | Code | Parse metadata; compute priority/escalation | AI Response Generator | Routing Decision |  |
| Routing Decision | IF | Decide escalation vs auto-email | Process & Analyze Response | Slack Escalation; Send Auto Response |  |
| Slack Escalation | HTTP Request | Slack escalation with details | Routing Decision (true) | Merge After Routing |  |
| Send Auto Response | Email Send | Send automated email reply | Routing Decision (false) | Merge After Routing |  |
| Merge After Routing | Merge | Re-joins escalation/auto-email for logging | Slack Escalation; Send Auto Response | Log to Analytics Dashboard |  |
| Log to Analytics Dashboard | Google Sheets | Append/update row for analytics | Merge After Routing | Save to Database |  |
| Save to Database | Postgres | Persist case record with upsert | Log to Analytics Dashboard | Send API Response |  |
| Send API Response | Respond to Webhook | Return structured JSON to caller | Save to Database | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add Webhook Trigger (Webhook node)**
   - HTTP Method: **POST**
   - Path: **customer-support**
   - Response Mode: **Using “Respond to Webhook” node**
   - (Optional) Document expected JSON body in the node notes.
3. **Add “Extract & Enrich Data” (Set node)**
   - Add fields:
     - `questionId` = `{{$now.toISO()}}-{{$json.body.email.split('@')[0]}}-{{Math.random().toString(36).substr(2, 9)}}`
     - `userQuestion` = `{{$json.body.question}}`
     - `userEmail` = `{{$json.body.email}}`
     - `userName` = `{{$json.body.name || 'Customer'}}`
     - `timestamp` = `{{$now.toISO()}}`
     - `conversationId` = `{{$json.body.conversationId || ('new-' + $now.toMillis())}}`
     - `userLanguage` = `{{$json.body.language || 'en'}}`
     - `priority` = `{{$json.body.priority || 'normal'}}`
   - Connect: **Webhook Trigger → Extract & Enrich Data**
4. **Add “Sentiment & Context Analysis” (Code node)**
   - Paste the JS that:
     - scores sentiment via keyword lists
     - detects urgency via urgent keywords
     - extracts `keywords`
     - detects `detectedLanguage` (ja/en)
     - outputs `requiresImmediateAttention`
   - Connect: **Extract & Enrich Data → Sentiment & Context Analysis**
5. **Add “Urgency Check” (IF node)**
   - Condition: Boolean equals **true**
   - Left value: `{{$json.requiresImmediateAttention}}`
   - Connect: **Sentiment & Context Analysis → Urgency Check**
6. **Add “Send Urgent Alert” (HTTP Request node)**
   - Method: **POST**
   - URL: `https://hooks.slack.com/services/YOUR_URGENT_WEBHOOK` (replace later)
   - Body: JSON (Slack Block Kit payload using n8n expressions)
   - Connect: **Urgency Check (true) → Send Urgent Alert**
7. **Add “Merge After Urgency” (Merge node)**
   - Keep default settings
   - Connect:
     - **Send Urgent Alert → Merge After Urgency (Input 1)**
     - **Urgency Check (false) → Merge After Urgency (Input 2)**
8. **Add “Fetch Conversation History” (Postgres node)**
   - Credentials: configure **PostgreSQL** connection (host/db/user/password/SSL as needed).
   - Operation: **Execute Query**
   - Query (adjust schema/table as needed):
     - Select recent history by conversation_id or user_email, limit 5.
   - Connect: **Merge After Urgency → Fetch Conversation History**
   - Note: if you don’t have PostgreSQL, disable this node and adjust downstream “Build AI Context” to not reference it.
9. **Add “Search Knowledge Base” (Code node)**
   - Implement mock results (as provided) or integrate a vector DB (Pinecone/Qdrant/Supabase).
   - Ensure output includes `searchResults: [{text, score, metadata}]`.
   - Connect: **Fetch Conversation History → Search Knowledge Base**
10. **Add “Build AI Context” (Code node)**
    - Build `knowledgeContext`, `historyContext`, `toneGuidance`, and `aiPrompt`.
    - If you keep the Postgres node, you can reference history using `$('Fetch Conversation History').all()`.
    - Connect: **Search Knowledge Base → Build AI Context**
11. **Add “AI Response Generator” (Code node)**
    - Either:
      - Keep mock generator that outputs `response` text containing the 3 metadata lines, or
      - Replace with an LLM node (OpenAI/Anthropic) and ensure the output format remains parseable (or update the parser).
    - Connect: **Build AI Context → AI Response Generator**
12. **Add “Process & Analyze Response” (Code node)**
    - Parse metadata via regex, compute `needsEscalation`, `priority`, `estimatedResponseTime`, and output a flattened record.
    - Connect: **AI Response Generator → Process & Analyze Response**
13. **Add “Routing Decision” (IF node)**
    - Condition: `{{$json.needsEscalation}}` is true
    - Connect: **Process & Analyze Response → Routing Decision**
14. **Add “Slack Escalation” (HTTP Request node)**
    - POST to `https://hooks.slack.com/services/YOUR_ESCALATION_WEBHOOK` (replace later)
    - JSON Slack Block Kit payload including priority, truncated response, links.
    - Connect: **Routing Decision (true) → Slack Escalation**
15. **Add “Send Auto Response” (Email Send node)**
    - Configure SMTP/Gmail credentials in n8n credentials.
    - To: `{{$json.userEmail}}`
    - From: `support@example.com` (replace with a verified sender)
    - Subject: language-aware expression (as in workflow)
    - **Important fix to make it functional:** set the **email body/text/html** to `{{$json.aiResponse}}` (or a formatted template).
    - Connect: **Routing Decision (false) → Send Auto Response**
16. **Add “Merge After Routing” (Merge node)**
    - Connect:
      - **Slack Escalation → Merge After Routing (Input 1)**
      - **Send Auto Response → Merge After Routing (Input 2)**
17. **Add “Log to Analytics Dashboard” (Google Sheets node)**
    - Credentials: Google OAuth2 with Sheets access.
    - Operation: **Append or Update**
    - Document ID: set your spreadsheet ID (replace `YOUR_SPREADSHEET_ID`)
    - Sheet: choose the target sheet (gid=0 if first sheet)
    - Matching: `questionId`
    - Map columns to the JSON fields (status/urgency/category/etc.).
    - Connect: **Merge After Routing → Log to Analytics Dashboard**
18. **Add “Save to Database” (Postgres node)**
    - Operation: Execute Query
    - Insert/upsert query into `support_conversations`
    - **Strongly recommended:** replace string interpolation with parameterized queries to avoid SQL injection and quoting issues.
    - Connect: **Log to Analytics Dashboard → Save to Database**
19. **Add “Send API Response” (Respond to Webhook node)**
    - Respond with JSON using expressions to include status/priority/message/metadata.
    - Connect: **Save to Database → Send API Response**
20. **(Optional) Add sticky notes**
    - Add sticky notes describing each block and setup steps (Slack webhooks, Sheets ID, email credentials, Postgres table, AI node replacement).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Replace `YOUR_URGENT_WEBHOOK` and `YOUR_ESCALATION_WEBHOOK` with real Slack Incoming Webhook URLs. | Slack Incoming Webhooks |
| Replace `YOUR_SPREADSHEET_ID` and authenticate Google Sheets in the Google Sheets node. | Google Sheets logging |
| Configure SMTP/Gmail credentials for the Email Send node; also set the email body (missing in the provided node configuration). | Email delivery |
| PostgreSQL is optional but DB nodes will fail unless `support_conversations` exists and matches the expected schema; use parameterized queries to avoid SQL injection. | Postgres integration |
| Mock AI nodes should be replaced with OpenAI/Anthropic (or similar) in production; ensure metadata format remains stable or update parsing logic. | AI integration |

Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.
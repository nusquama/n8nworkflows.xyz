Draft and review grounded support email replies with Gmail, OpenAI and Claude

https://n8nworkflows.xyz/workflows/draft-and-review-grounded-support-email-replies-with-gmail--openai-and-claude-18957


# Draft and review grounded support email replies with Gmail, OpenAI and Claude

### 1. Workflow Overview

This workflow is a comprehensive, multi-vendor AI-powered support desk automation pipeline. It ingests and chunks documentation from Google Drive into a PostgreSQL database with `pgvector`, handles inbound customer support requests from both Gmail and an external webhook, protects Personally Identifiable Information (PII) before any model processing, drafts grounded responses using retrieved documentation, evaluates those responses through an independent LLM critic quality gate, routes messages for human approval or escalation as necessary, and automatically compiles weekly operational metrics.

The logic is partitioned into three main functional blocks:

- **1.1 Knowledge Base Ingestion:** Pulls files from a Google Drive folder, compares versions against a PostgreSQL ledger to process only new or modified documents, purges outdated vector chunks, splits and embeds text using OpenAI, and records ingestion metadata.
- **1.2 Reply Pipeline & Quality Gate:** Normalizes incoming tickets from Gmail or a webhook, masks sensitive PII, classifies requests, applies business policy rules via a router, generates grounded answers using vector search tools, evaluates drafts with a secondary LLM critic, applies a strict quality gate, delivers approved responses via Gmail, handles human-in-the-loop approvals, and manages escalations.
- **1.3 Weekly Metrics Digest:** Runs on a scheduled cron trigger to analyze the previous week's performance data from PostgreSQL, aggregates resolution statistics, knowledge base gaps, and escalation categories, and emails a structured operational summary.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Knowledge Base Ingestion
- **Overview:** Keeps the vector knowledge base synchronized with a designated Google Drive folder. It identifies updated or new documents, prevents duplicate versions by purging stale chunks, embeds text into PostgreSQL using `pgvector`, and updates an ingestion tracking ledger.
- **Nodes Involved:** 
  - `Ingest On Demand`
  - `Ingest Nightly 02:00`
  - `List KB Files`
  - `Fetch Ingest Ledger`
  - `Select New Or Changed`
  - `Download KB File`
  - `Purge Old Chunks`
  - `KB Embeddings`
  - `KB Splitter`
  - `KB Loader`
  - `Embed Into PGVector`
  - `Record In Ledger`

- **Node Details:**
  - **Ingest On Demand** (`n8n-nodes-base.manualTrigger`)
    - *Role:* Provides a manual trigger to run the ingestion segment on demand.
    - *Configuration:* Default.
    - *Connections:* Input: None; Output: `List KB Files`.
    - *Edge Cases:* None.
  - **Ingest Nightly 02:00** (`n8n-nodes-base.scheduleTrigger`)
    - *Role:* Triggers knowledge base synchronization automatically every night at 02:00 AM.
    - *Configuration:* Schedule rule set to run at hour 2.
    - *Connections:* Input: None; Output: `List KB Files`.
    - *Edge Cases:* None.
  - **List KB Files** (`n8n-nodes-base.googleDrive`)
    - *Role:* Retrieves a complete file listing from a target Google Drive folder excluding folders and trashed items.
    - *Configuration:* Resource: `fileFolder`, Operation: `fileFolder`, Query string filtering out folders, returns all items including file metadata (`id`, `name`, `mimeType`, `webViewLink`, `version`).
    - *Credentials:* Google Drive OAuth2 API.
    - *Connections:* Input: `Ingest On Demand`, `Ingest Nightly 02:00`; Output: `Fetch Ingest Ledger`.
    - *Edge Cases:* API rate limits, revoked credentials, or incorrect folder ID (`KB_FOLDER_ID`).
  - **Fetch Ingest Ledger** (`n8n-nodes-base.postgres`)
    - *Role:* Queries the PostgreSQL tracking table to retrieve existing document IDs and version numbers.
    - *Configuration:* Operation: `executeQuery`, Query: `select doc_id, doc_version from support_kb_ledger`.
    - *Credentials:* Postgres account.
    - *Connections:* Input: `List KB Files`; Output: `Select New Or Changed`.
    - *Edge Cases:* Database connection failure, missing `support_kb_ledger` table.
  - **Select New Or Changed** (`n8n-nodes-base.code`)
    - *Role:* Compares Google Drive file items against the PostgreSQL ledger to filter out unchanged files and save embedding costs.
    - *Configuration:* JavaScript mapping and filtering logic detecting `new` or `changed` files.
    - *Connections:* Input: `Fetch Ingest Ledger`; Output: `Download KB File`.
    - *Edge Cases:* Null values in version fields or missing file metadata properties.
  - **Download KB File** (`n8n-nodes-base.googleDrive`)
    - *Role:* Downloads the binary content of new or modified files, converting Google Docs formats to plain text.
    - *Configuration:* File ID mapped from item expression, binary property name `data`, automatic conversion parameters.
    - *Credentials:* Google Drive OAuth2 API.
    - *Connections:* Input: `Select New Or Changed`; Output: `Purge Old Chunks`.
    - *Edge Cases:* Large file sizes, unsupported MIME types, download timeouts.
  - **Purge Old Chunks** (`n8n-nodes-base.postgres`)
    - *Role:* Deletes existing vector entries for a document before new chunks are inserted to prevent duplicate versions.
    - *Configuration:* Operation: `executeQuery`, Query: `delete from support_kb_vectors where metadata->>'doc_id' = $1`, query replacement parameter mapped to current document ID.
    - *Credentials:* Postgres account.
    - *Connections:* Input: `Download KB File`; Output: `Embed Into PGVector`.
    - *Edge Cases:* Database query timeout or structural mismatch in JSONB metadata path.
  - **KB Embeddings** (`@n8n/n8n-nodes-langchain.embeddingsOpenAi`)
    - *Role:* Generates vector embeddings for text chunks.
    - *Configuration:* Model configured with `dimensions: 1536` (`text-embedding-3-small`).
    - *Credentials:* OpenAI API.
    - *Connections:* Input: None (linked to `Embed Into PGVector` as AI embedding provider); Output: `Embed Into PGVector`.
    - *Edge Cases:* API authentication errors, rate limiting, token limits.
  - **KB Splitter** (`@n8n/n8n-nodes-langchain.textSplitterRecursiveCharacterTextSplitter`)
    - *Role:* Splits document text into manageable chunks.
    - *Configuration:* Chunk size: 1200, Chunk overlap: 150.
    - *Connections:* Input: None; Output: `KB Loader`.
    - *Edge Cases:* Extremely short or empty documents resulting in zero chunks.
  - **KB Loader** (`@n8n/n8n-nodes-langchain.documentDefaultDataLoader`)
    - *Role:* Loads documents into memory while injecting metadata (doc ID, title, URL) into each chunk.
    - *Configuration:* Binary data input, custom text splitting mode, dynamic metadata assignment expressions.
    - *Connections:* Input: `KB Splitter`; Output: `Embed Into PGVector`.
    - *Edge Cases:* Missing binary data properties.
  - **Embed Into PGVector** (`@n8n/n8n-nodes-langchain.vectorStorePGVector`)
    - *Role:* Stores embedded document chunks and metadata into the PostgreSQL pgvector store.
    - *Configuration:* Mode: `insert`, Target table: `support_kb_vectors`.
    - *Credentials:* Postgres account.
    - *Connections:* Input: `Purge Old Chunks`, `KB Embeddings`, `KB Loader`; Output: `Record In Ledger`.
    - *Edge Cases:* Dimension mismatch (must match 1536), database write failures.
  - **Record In Ledger** (`n8n-nodes-base.postgres`)
    - *Role:* Inserts or updates ingestion records in the ledger table for successfully processed documents.
    - *Configuration:* Operation: `executeQuery`, UPSERT query updating `doc_title`, `doc_version`, `doc_url`, and `ingested_at`.
    - *Credentials:* Postgres account.
    - *Connections:* Input: `Embed Into PGVector`; Output: None (End of Block 1.1).
    - *Edge Cases:* Unique constraint violations, connection failures.

---

#### Block 1.2: Reply Pipeline & Quality Gate
- **Overview:** Ingests tickets from Gmail or a webhook, normalizes the payload, masks PII, classifies the request using an LLM, applies policy routing, generates a grounded answer via vector search, evaluates the draft using a critic LLM, enforces quality thresholds, and delivers the response either automatically or via human approval/escalation queues.
- **Nodes Involved:**
  - `Ticket From Gmail`
  - `Ticket From Helpdesk`
  - `Normalize Ticket`
  - `Mask PII`
  - `Classifier Model`
  - `Classification Schema`
  - `Classify Ticket`
  - `Assemble Ticket Record`
  - `Log Ticket`
  - `Policy Router`
  - `Log Suppressed`
  - `Drafting Model`
  - `Answer Embeddings`
  - `Search Knowledge Base`
  - `Draft Schema`
  - `Draft Grounded Answer`
  - `Critic Model`
  - `Critic Schema`
  - `Critic Review`
  - `Merge Draft And Verdict`
  - `Quality Gate`
  - `Unmask Answer`
  - `Choose Delivery Channel`
  - `Reply In Gmail Thread`
  - `Send Email Reply`
  - `Log Sent Reply`
  - `Request Human Approval`
  - `Approval Granted`
  - `Build Escalation Packet`
  - `Escalate To Human Queue`
  - `Log Escalation`

- **Node Details:**
  - **Ticket From Gmail** (`n8n-nodes-base.gmailTrigger`)
    - *Role:* Triggers on incoming unread emails matching the support label filter.
    - *Configuration:* Filters set to `label:support`, `readStatus:unread`, polling every few minutes.
    - *Credentials:* Gmail OAuth2.
    - *Connections:* Input: None; Output: `Normalize Ticket`.
    - *Edge Cases:* Auth token expiration, rate limiting.
  - **Ticket From Helpdesk** (`n8n-nodes-base.webhook`)
    - *Role:* Receives external helpdesk ticket payloads via HTTP POST.
    - *Configuration:* Path: `support-ticket`, Method: `POST`, Header Authentication.
    - *Connections:* Input: None; Output: `Normalize Ticket`.
    - *Edge Cases:* Invalid JSON payloads, missing authentication headers.
  - **Normalize Ticket** (`n8n-nodes-base.code`)
    - *Role:* Flattens Gmail and webhook payloads into a unified ticket data structure and truncates raw body text to 8,000 characters.
    - *Configuration:* JavaScript transformation extracting sender email, subject, message ID, thread ID, and body.
    - *Connections:* Input: `Ticket From Gmail`, `Ticket From Helpdesk`; Output: `Mask PII`.
    - *Edge Cases:* Malformed webhook payloads missing standard fields.
  - **Mask PII** (`n8n-nodes-base.code`)
    - *Role:* Scans the ticket body using regex rules to replace sensitive information (emails, IBANs, credit cards, phone numbers) with placeholder tokens.
    - *Configuration:* JavaScript regex replacement logic maintaining a bidirectional token-to-value map (`pii_map`).
    - *Connections:* Input: `Normalize Ticket`; Output: `Classify Ticket`.
    - *Edge Cases:* Overlapping regex patterns or unformatted edge-case phone numbers.
  - **Classifier Model** (`@n8n/n8n-nodes-langchain.lmChatOpenRouter`)
    - *Role:* Provides the underlying OpenRouter chat model for ticket classification.
    - *Configuration:* Model: `openai/gpt-5.6-luna`, Temperature: 0, Max retries: 2.
    - *Credentials:* OpenRouter Mina.
    - *Connections:* Input: None; Output: `Classify Ticket`.
    - *Edge Cases:* Model API downtime, rate limits.
  - **Classification Schema** (`@n8n/n8n-nodes-langchain.outputParserStructured`)
    - *Role:* Enforces a structured JSON output schema for ticket classification results.
    - *Configuration:* Manual schema defining category enum, intent, urgency, sentiment, language code, answerability boolean, and policy class.
    - *Connections:* Input: None; Output: `Classify Ticket`.
    - *Edge Cases:* Model output failing schema validation.
  - **Classify Ticket** (`@n8n/n8n-nodes-langchain.chainLlm`)
    - *Role:* Analyzes the masked ticket text and outputs structured classification metadata.
    - *Configuration:* Prompt defining policy rules (drop, restricted, auto-answerable) and language detection.
    - *Connections:* Input: `Mask PII`, `Classifier Model`, `Classification Schema`; Output: `Assemble Ticket Record`.
    - *Edge Cases:* LLM classification errors or hallucinations.
  - **Assemble Ticket Record** (`n8n-nodes-base.code`)
    - *Role:* Merges LLM classification outputs with the original masked ticket record into a single consolidated item.
    - *Configuration:* JavaScript execution per item mapping parsed fields with fail-closed defaults.
    - *Connections:* Input: `Classify Ticket`; Output: `Log Ticket`, `Policy Router`.
    - *Edge Cases:* Missing model output property.
  - **Log Ticket** (`n8n-nodes-base.postgres`)
    - *Role:* Persists incoming ticket metadata and classification details to PostgreSQL.
    - *Configuration:* Operation: `executeQuery`, UPSERT query on `support_tickets`.
    - *Credentials:* Postgres account.
    - *Connections:* Input: `Assemble Ticket Record`; Output: None.
    - *Edge Cases:* Database constraint violations.
  - **Policy Router** (`n8n-nodes-base.switch`)
    - *Role:* Routes tickets based on policy classification and doc-answerability flags.
    - *Configuration:* Rules evaluating `policy_class == 'auto_answerable'` and `is_answerable_from_docs == true` (output: `auto_answerable`), `policy_class == 'drop'` (output: `drop`), with fallback output routing to `restricted`.
    - *Connections:* Input: `Assemble Ticket Record`; Output: `Draft Grounded Answer` (auto-answerable), `Log Suppressed` (drop), `Build Escalation Packet` (restricted/fallback).
    - *Edge Cases:* Unexpected policy classification values.
  - **Log Suppressed** (`n8n-nodes-base.postgres`)
    - *Role:* Records suppressed or dropped junk tickets in the answers table.
    - *Configuration:* Operation: `executeQuery`, Query: `insert into support_answers (ticket_ref, outcome, critic_verdict) values ($1, 'suppressed', 'not_a_ticket')`.
    - *Credentials:* Postgres account.
    - *Connections:* Input: `Policy Router`; Output: None.
    - *Edge Cases:* Database write failure.
  - **Drafting Model** (`@n8n/n8n-nodes-langchain.lmChatOpenRouter`)
    - *Role:* Provides the chat model (Claude) for generating grounded support responses.
    - *Configuration:* Model: `anthropic/claude-opus-5`, Temperature: 0.2, Max retries: 2.
    - *Credentials:* OpenRouter Mina.
    - *Connections:* Input: None; Output: `Draft Grounded Answer`.
    - *Edge Cases:* API errors or context window limits.
  - **Answer Embeddings** (`@n8n/n8n-nodes-langchain.embeddingsOpenAi`)
    - *Role:* Generates query embeddings for vector knowledge base retrieval.
    - *Configuration:* Dimensions: 1536 (`text-embedding-3-small`).
    - *Credentials:* OpenAI API.
    - *Connections:* Input: None; Output: `Search Knowledge Base`.
    - *Edge Cases:* API rate limits.
  - **Search Knowledge Base** (`@n8n/n8n-nodes-langchain.vectorStorePGVector`)
    - *Role:* Acts as a retrieval tool enabling the drafting agent to search the pgvector knowledge base.
    - *Configuration:* Mode: `retrieve-as-tool`, Top K: 6, Distance strategy: `cosine`, Table: `support_kb_vectors`.
    - *Credentials:* Postgres account.
    - *Connections:* Input: `Answer Embeddings`; Output: `Draft Grounded Answer` (AI tool connection).
    - *Edge Cases:* Empty retrieval results or database connection failure.
  - **Draft Schema** (`@n8n/n8n-nodes-langchain.outputParserStructured`)
    - *Role:* Enforces structured output formatting for the drafting agent.
    - *Configuration:* Manual schema defining markdown answer, citations array, confidence score, unsupported claims array, and KB gap note.
    - *Connections:* Input: None; Output: `Draft Grounded Answer`.
    - *Edge Cases:* Malformed JSON output from the LLM.
  - **Draft Grounded Answer** (`@n8n/n8n-nodes-langchain.agent`)
    - *Role:* AI agent that executes knowledge base searches and drafts a grounded response with citations.
    - *Configuration:* Prompt instructing strict adherence to retrieved documents, handling up to 8 iterations, structured output parser. On error continues to error output.
    - *Connections:* Input: `Policy Router`, `Drafting Model`, `Search Knowledge Base`, `Draft Schema`; Output: `Critic Review`, `Build Escalation Packet` (on error).
    - *Edge Cases:* Maximum iterations reached without resolution, ungrounded drafting.
  - **Critic Model** (`@n8n/n8n-nodes-langchain.lmChatOpenRouter`)
    - *Role:* Provides the evaluation chat model for reviewing draft answers.
    - *Configuration:* Model: `openai/gpt-5.6-luna`, Temperature: 0, Max retries: 2.
    - *Credentials:* OpenRouter Mina.
    - *Connections:* Input: None; Output: `Critic Review`.
    - *Edge Cases:* API timeouts or failures.
  - **Critic Schema** (`@n8n/n8n-nodes-langchain.outputParserStructured`)
    - *Role:* Enforces structured output formatting for the critic evaluation model.
    - *Configuration:* Manual schema defining accuracy, groundedness, tone, completeness, overall score, verdict enum (`send`, `revise`, `escalate`), and issues array.
    - *Connections:* Input: None; Output: `Critic Review`.
    - *Edge Cases:* Invalid schema parsing.
  - **Critic Review** (`@n8n/n8n-nodes-langchain.chainLlm`)
    - *Role:* Critically evaluates the draft answer against citations, checking for groundedness and policy compliance.
    - *Configuration:* Prompt instructing adversarial review, penalizing ungrounded claims, structured output parser.
    - *Connections:* Input: `Draft Grounded Answer`, `Critic Model`, `Critic Schema`; Output: `Merge Draft And Verdict`.
    - *Edge Cases:* Critic hallucination or lenient scoring.
  - **Merge Draft And Verdict** (`n8n-nodes-base.code`)
    - *Role:* Combines ticket data, draft content, citations, and critic scores into a single unified record for quality gate evaluation.
    - *Configuration:* JavaScript execution per item safely handling arrays and numeric parsing.
    - *Connections:* Input: `Critic Review`; Output: `Quality Gate`.
    - *Edge Cases:* Missing nested properties.
  - **Quality Gate** (`n8n-nodes-base.if`)
    - *Role:* Evaluates whether a draft meets all strict criteria required for automatic delivery.
    - *Configuration:* Conditions (AND logic): `critic_overall >= 8`, `critic_verdict == 'send'`, `draft_confidence >= 0.7`, `citation_count >= 1`, `unsupported_count == 0`.
    - *Connections:* Input: `Merge Draft And Verdict`; Output: `Unmask Answer` (true branch), `Request Human Approval` (false branch).
    - *Edge Cases:* Strict thresholds causing high escalation volume if documentation is thin.
  - **Unmask Answer** (`n8n-nodes-base.code`)
    - *Role:* Restores original PII data (emails, credit cards, etc.) into the drafted answer text using the stored map, and formats sources as HTML.
    - *Configuration:* JavaScript token replacement supporting both auto-send and human-approved pathways.
    - *Connections:* Input: `Quality Gate`, `Approval Granted`; Output: `Choose Delivery Channel`.
    - *Edge Cases:* Missing PII map tokens or malformed HTML formatting.
  - **Choose Delivery Channel** (`n8n-nodes-base.if`)
    - *Role:* Determines whether the response should be sent as a Gmail thread reply or a new email based on the original channel.
    - *Configuration:* Condition: `channel == 'gmail'`.
    - *Connections:* Input: `Unmask Answer`; Output: `Reply In Gmail Thread` (true), `Send Email Reply` (false/helpdesk).
    - *Edge Cases:* Missing thread ID for Gmail channels.
  - **Reply In Gmail Thread** (`n8n-nodes-base.gmail`)
    - *Role:* Sends the final HTML response as a reply within an existing Gmail thread.
    - *Configuration:* Operation: `reply`, Message ID mapped from ticket, options set to `replyToSenderOnly` and no attribution.
    - *Credentials:* Gmail OAuth2.
    - *Connections:* Input: `Choose Delivery Channel`; Output: `Log Sent Reply`.
    - *Edge Cases:* Gmail API rate limits, invalid message ID.
  - **Send Email Reply** (`n8n-nodes-base.gmail`)
    - *Role:* Sends the final response as a new email message to helpdesk tickets or external inquiries.
    - *Configuration:* Operation: `send`, Recipient: `from_email`, Subject: `Re: [subject]`, message body HTML.
    - *Credentials:* Gmail OAuth2.
    - *Connections:* Input: `Choose Delivery Channel`; Output: `Log Sent Reply`.
    - *Edge Cases:* Invalid recipient address.
  - **Log Sent Reply** (`n8n-nodes-base.postgres`)
    - *Role:* Records successful automated or human-approved sent replies in PostgreSQL.
    - *Configuration:* Operation: `executeQuery`, Query inserting answers, draft details, citations, scores, outcomes, and reviewer info into `support_answers`.
    - *Credentials:* Postgres account.
    - *Connections:* Input: `Reply In Gmail Thread`, `Send Email Reply`; Output: None.
    - *Edge Cases:* Database insertion errors.
  - **Request Human Approval** (`n8n-nodes-base.gmail`)
    - *Role:* Sends an interactive approval email to the support lead containing the ticket, draft, critic objections, and review options.
    - *Configuration:* Operation: `sendAndWait`, Recipient: `SUPPORT_LEAD_EMAIL`, double approval options (`Send this reply` vs `I will write it myself`), wait timeout limit.
    - *Credentials:* Gmail OAuth2.
    - *Connections:* Input: `Quality Gate`; Output: `Approval Granted`.
    - *Edge Cases:* Email delivery failure, timeout waiting for reviewer action.
  - **Approval Granted** (`n8n-nodes-base.if`)
    - *Role:* Checks whether the human reviewer approved the draft via email.
    - *Configuration:* Loose type validation checking `data.approved == true`.
    - *Connections:* Input: `Request Human Approval`; Output: `Unmask Answer` (true/approved), `Build Escalation Packet` (false/declined).
    - *Edge Cases:* Reviewer clicks decline or request times out.
  - **Build Escalation Packet** (`n8n-nodes-base.code`)
    - *Role:* Assembles escalation details for restricted tickets, declined drafts, or errored agent runs, restoring PII for the human handler.
    - *Configuration:* JavaScript try/catch block extracting draft data, issues, and restoring PII tokens.
    - *Connections:* Input: `Draft Grounded Answer`, `Approval Granted`, `Policy Router`; Output: `Escalate To Human Queue`.
    - *Edge Cases:* Missing record properties.
  - **Escalate To Human Queue** (`n8n-nodes-base.gmail`)
    - *Role:* Forwards the escalation ticket packet and drafting notes to the support lead via email.
    - *Configuration:* Operation: `send`, Recipient: `SUPPORT_LEAD_EMAIL`, Reply-To set to customer email, subject prefix `Escalated`.
    - *Credentials:* Gmail OAuth2.
    - *Connections:* Input: `Build Escalation Packet`; Output: `Log Escalation`.
    - *Edge Cases:* Gmail API delivery failure.
  - **Log Escalation** (`n8n-nodes-base.postgres`)
    - *Role:* Logs ticket escalation outcomes and reviewer objections in PostgreSQL.
    - *Configuration:* Operation: `executeQuery`, Query inserting escalation records into `support_answers`.
    - *Credentials:* Postgres account.
    - *Connections:* Input: `Escalate To Human Queue`; Output: None.
    - *Edge Cases:* Database connection timeout.

---

#### Block 1.3: Weekly Metrics Digest
- **Overview:** Runs weekly to query operational support metrics and knowledge base gaps from PostgreSQL, formats them into a clean HTML email digest, and sends it to the support lead.
- **Nodes Involved:**
  - `Weekly Metrics Cron`
  - `Query Reply Metrics`
  - `Format Metrics Email`
  - `Send Metrics Digest`

- **Node Details:**
  - **Weekly Metrics Cron** (`n8n-nodes-base.scheduleTrigger`)
    - *Role:* Triggers the weekly metrics workflow every Monday morning at 07:00 AM.
    - *Configuration:* Schedule interval set for Mondays at hour 7.
    - *Connections:* Input: None; Output: `Query Reply Metrics`.
    - *Edge Cases:* None.
  - **Query Reply Metrics** (`n8n-nodes-base.postgres`)
    - *Role:* Queries PostgreSQL for support answer outcomes, auto-answer rates, average critic scores, KB gaps, and escalation categories over the past 7 days.
    - *Configuration:* Operation: `executeQuery`, Complex SQL query using CTEs and JSON aggregations.
    - *Credentials:* Postgres account.
    - *Connections:* Input: `Weekly Metrics Cron`; Output: `Format Metrics Email`.
    - *Edge Cases:* Database query timeout or empty dataset resulting in zero counts.
  - **Format Metrics Email** (`n8n-nodes-base.code`)
    - *Role:* Transforms metric query results into a structured HTML email digest including KB gaps and escalation breakdown.
    - *Configuration:* JavaScript string building generating an HTML metrics report table and bullet lists.
    - *Connections:* Input: `Query Reply Metrics`; Output: `Send Metrics Digest`.
    - *Edge Cases:* Null values in numeric aggregations.
  - **Send Metrics Digest** (`n8n-nodes-base.gmail`)
    - *Role:* Emails the weekly operational metrics digest to the support lead.
    - *Configuration:* Operation: `send`, Recipient: `SUPPORT_LEAD_EMAIL`, Subject: `Support replies weekly · [auto_answer_rate]% answered automatically`.
    - *Credentials:* Gmail OAuth2.
    - *Connections:* Input: `Format Metrics Email`; Output: None.
    - *Edge Cases:* Gmail delivery failure or invalid recipient address.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| README | n8n-nodes-base.stickyNote | Documentation and setup instructions | None | None | # Support Reply Agent<br><br>One canvas, three segments, one import. It answers the tickets it can prove an<br>answer for, and escalates the ones it cannot.<br><br>**The idea:** a drafting agent writes a grounded answer out of your own docs,<br>then a **second model from a different vendor** grades that answer before<br>anything is sent. Only drafts that survive the grader go out on their own.<br>Everything else reaches a human with the draft and the grader's objections<br>attached, so reviewing is faster than writing from scratch.<br><br>**Segments**<br>- **A** Knowledge Base Ingest, Drive folder to pgvector, change-detected<br>- **B** Reply Pipeline, ticket in, cited reply or escalation out<br>- **C** Weekly Metrics, auto-answer rate, escalations, and the KB gaps behind them<br><br>**Before the first run**<br>1. Run `sql/01-schema.sql` against your Postgres<br>2. Put your Drive folder id in *List KB Files*, replacing `KB_FOLDER_ID`<br>3. Put your reviewer address in *Request Human Approval*, *Escalate To Human Queue*<br>   and *Send Metrics Digest*, replacing `SUPPORT_LEAD_EMAIL`<br>4. Run segment A once, so the vector store is not empty<br>5. Run segment B by hand on one ticket before you activate anything<br><br>**It is deliberately conservative.** An empty knowledge base produces zero<br>citations, zero citations fail the gate, and a failed gate escalates. It does<br>not guess when it has nothing to cite. |
| Segment A Frame | n8n-nodes-base.stickyNote | Visual container for KB Ingest segment | None | None | ## A · Knowledge Base Ingest<br>### Drive folder, chunks, pgvector, with change detection<br><br>Reads the ledger first and only re-embeds files whose Drive version moved. Old<br>chunks for a changed document are deleted before the new ones land, so a<br>re-ingest replaces instead of duplicating. That duplication is the usual reason<br>a RAG bot starts citing two different versions of the same policy.<br><br>Embeddings are OpenAI `text-embedding-3-small` at 1536 dimensions. Change the<br>model and you must drop `support_kb_vectors`, because the column width has to<br>match the model.<br><br>Runs nightly at 02:00, or on demand. |
| Segment B Frame | n8n-nodes-base.stickyNote | Visual container for Reply Pipeline segment | None | None | ## B · Reply Pipeline<br>### Two ways in, classify, policy, draft, critic, gate, deliver or escalate<br><br>**PII is masked before any model sees the ticket** and restored only on the way<br>back out, so email addresses and card numbers never reach a model vendor.<br><br>**Policy routing runs before the model spends anything.** Billing, refunds,<br>security and legal never auto-answer, no matter how confident the draft looks.<br>That is a business rule, not a model decision, so it lives in a Switch.<br><br>**The gate is an AND of five conditions, not one score:** critic overall at<br>least 8, verdict = send, drafter confidence at least 0.7, at least one citation,<br>and zero unsupported claims. Any single failure routes to a human. |
| Segment C Frame | n8n-nodes-base.stickyNote | Visual container for Weekly Metrics segment | None | None | ## C · Weekly Metrics<br>### Auto-answer rate, escalation reasons, and the KB gaps behind them<br><br>Mondays 07:00. The column worth reading is not the auto-answer rate, it is<br>`kb_gaps`: that is the drafting agent telling you, in its own words, which<br>documents you have not written yet. |
| Ingest On Demand | n8n-nodes-base.manualTrigger | Manual trigger for KB ingestion | None | List KB Files | |
| Ingest Nightly 02:00 | n8n-nodes-base.scheduleTrigger | Scheduled nightly trigger for KB ingestion | None | List KB Files | |
| List KB Files | n8n-nodes-base.googleDrive | Lists files in Google Drive KB folder | Ingest On Demand, Ingest Nightly 02:00 | Fetch Ingest Ledger | |
| Fetch Ingest Ledger | n8n-nodes-base.postgres | Queries PostgreSQL ingest ledger | List KB Files | Select New Or Changed | |
| Select New Or Changed | n8n-nodes-base.code | Compares Drive files against ledger | Fetch Ingest Ledger | Download KB File | |
| Download KB File | n8n-nodes-base.googleDrive | Downloads changed KB files | Select New Or Changed | Purge Old Chunks | |
| Purge Old Chunks | n8n-nodes-base.postgres | Deletes old vector chunks for updated doc | Download KB File | Embed Into PGVector | |
| Embed Into PGVector | @n8n/n8n-nodes-langchain.vectorStorePGVector | Inserts document embeddings into pgvector | Download KB File, KB Embeddings, KB Loader | Record In Ledger | |
| KB Embeddings | @n8n/n8n-nodes-langchain.embeddingsOpenAi | Generates OpenAI embeddings | None | Embed Into PGVector | |
| KB Loader | @n8n/n8n-nodes-langchain.documentDefaultDataLoader | Loads and annotates documents | KB Splitter | Embed Into PGVector | |
| KB Splitter | @n8n/n8n-nodes-langchain.textSplitterRecursiveCharacterTextSplitter | Splits KB text into chunks | None | KB Loader | |
| Record In Ledger | n8n-nodes-base.postgres | Records ingestion state in ledger | Embed Into PGVector | None | |
| Ticket From Gmail | n8n-nodes-base.gmailTrigger | Triggers on unread support emails | None | Normalize Ticket | |
| Ticket From Helpdesk | n8n-nodes-base.webhook | Receives helpdesk tickets via webhook | None | Normalize Ticket | |
| Normalize Ticket | n8n-nodes-base.code | Normalizes triggers into a unified ticket schema | Ticket From Gmail, Ticket From Helpdesk | Mask PII | |
| Mask PII | n8n-nodes-base.code | Redacts sensitive PII from ticket bodies | Normalize Ticket | Classify Ticket | |
| Classifier Model | @n8n/n8n-nodes-langchain.lmChatOpenRouter | Chat model for ticket classification | None | Classify Ticket | |
| Classification Schema | @n8n/n8n-nodes-langchain.outputParserStructured | Structured output parser for classification | None | Classify Ticket | |
| Classify Ticket | @n8n/n8n-nodes-langchain.chainLlm | Classifies support tickets | Mask PII, Classifier Model, Classification Schema | Assemble Ticket Record | |
| Assemble Ticket Record | n8n-nodes-base.code | Merges classification with ticket data | Classify Ticket | Log Ticket, Policy Router | |
| Log Ticket | n8n-nodes-base.postgres | Logs ticket record to database | Assemble Ticket Record | None | |
| Policy Router | n8n-nodes-base.switch | Routes tickets based on policy and answerability | Assemble Ticket Record | Draft Grounded Answer, Log Suppressed, Build Escalation Packet | |
| Log Suppressed | n8n-nodes-base.postgres | Logs suppressed junk tickets | Policy Router | None | |
| Drafting Model | @n8n/n8n-nodes-langchain.lmChatOpenRouter | Chat model for drafting answers | None | Draft Grounded Answer | |
| Answer Embeddings | @n8n/n8n-nodes-langchain.embeddingsOpenAi | Generates embeddings for KB search tool | None | Search Knowledge Base | |
| KB Search Tool | @n8n/n8n-nodes-langchain.vectorStorePGVector | Vector search tool for the drafting agent | Answer Embeddings | Draft Grounded Answer | |
| Draft Schema | @n8n/n8n-nodes-langchain.outputParserStructured | Structured output parser for answers | None | Draft Grounded Answer | |
| Draft Grounded Answer | @n8n/n8n-nodes-langchain.agent | AI agent generating cited answers | Policy Router, Drafting Model, KB Search Tool, Draft Schema | Critic Review, Build Escalation Packet | |
| Critic Model | @n8n/n8n-nodes-langchain.lmChatOpenRouter | Chat model for answer evaluation | None | Critic Review | |
| Critic Schema | @n8n/n8n-nodes-langchain.outputParserStructured | Structured output parser for critic review | None | Critic Review | |
| Critic Review | @n8n/n8n-nodes-langchain.chainLlm | Critically reviews draft answers | Draft Grounded Answer, Critic Model, Critic Schema | Merge Draft And Verdict | |
| Merge Draft And Verdict | n8n-nodes-base.code | Combines draft, ticket, and critic results | Critic Review | Quality Gate | |
| Quality Gate | n8n-nodes-base.if | Evaluates quality thresholds for auto-delivery | Merge Draft And Verdict | Unmask Answer, Request Human Approval | |
| Unmask Answer | n8n-nodes-base.code | Restores PII tokens and formats HTML output | Quality Gate, Approval Granted | Choose Delivery Channel | |
| Choose Delivery Channel | n8n-nodes-base.if | Routes delivery by channel type | Unmask Answer | Reply In Gmail Thread, Send Email Reply | |
| Reply In Gmail Thread | n8n-nodes-base.gmail | Replies within existing Gmail thread | Choose Delivery Channel | Log Sent Reply | |
| Send Email Reply | n8n-nodes-base.gmail | Sends new email reply | Choose Delivery Channel | Log Sent Reply | |
| Log Sent Reply | n8n-nodes-base.postgres | Logs sent replies in database | Reply In Gmail Thread, Send Email Reply | None | |
| Request Human Approval | n8n-nodes-base.gmail | Sends interactive approval request to support lead | Quality Gate | Approval Granted | |
| Approval Granted | n8n-nodes-base.if | Checks if human reviewer approved draft | Request Human Approval | Unmask Answer, Build Escalation Packet | |
| Build Escalation Packet | n8n-nodes-base.code | Assembles escalation data and restores PII | Draft Grounded Answer, Approval Granted, Policy Router | Escalate To Human Queue | |
| Escalate To Human Queue | n8n-nodes-base.gmail | Emails escalation details to support lead | Build Escalation Packet | Log Escalation | |
| Log Escalation | n8n-nodes-base.postgres | Logs ticket escalation in database | Escalate To Human Queue | None | |
| Weekly Metrics Cron | n8n-nodes-base.scheduleTrigger | Scheduled weekly trigger for metrics | None | Query Reply Metrics | |
| Query Reply Metrics | n8n-nodes-base.postgres | Queries operational metrics from database | Weekly Metrics Cron | Format Metrics Email | |
| Format Metrics Email | n8n-nodes-base.code | Formats metrics report into HTML | Query Reply Metrics | Send Metrics Digest | |
| Send Metrics Digest | n8n-nodes-base.gmail | Emails weekly metrics digest to support lead | Format Metrics Email | None | |

---

### 4. Reproducing the Workflow from Scratch

To rebuild this workflow manually in n8n, execute the following steps:

1. **Database Setup:** Run the required schema creation SQL (`sql/01-schema.sql`) on your PostgreSQL instance, ensuring `pgvector` extension is enabled and the `support_kb_vectors` embedding dimension column matches `1536`.
2. **Create Segment A (Knowledge Base Ingest):**
   - Place a `Manual Trigger` (`ingest-manual`) and a `Schedule Trigger` (`ingest-cron`, interval: 2 hours).
   - Add a `Google Drive` node (`kb-list`) configured to list files in folder ID `KB_FOLDER_ID`.
   - Add a `Postgres` node (`kb-ledger`) to query `support_kb_ledger`.
   - Add a `Code` node (`kb-diff`) to compare Drive versions with the ledger and filter new or changed files.
   - Add a `Google Drive` node (`kb-download`) to download file binaries (`data`).
   - Add a `Postgres` node (`kb-purge`) to delete stale vector chunks where `metadata->>'doc_id' = $1`.
   - Add an `OpenAI Embeddings` node (`kb-embeddings`, dimensions: 1536) and a `Recursive Character Text Splitter` (`kb-splitter`, chunk size: 1200, overlap: 150) connected to a `Default Document Loader` (`kb-loader`).
   - Connect the embeddings, loader, and purge output to a `Postgres Vector Store` (`kb-embed`) targeting table `support_kb_vectors`.
   - Add a `Postgres` node (`kb-record`) to UPSERT ingestion state into `support_kb_ledger`.
3. **Create Segment B (Reply Pipeline & Quality Gate):**
   - Add a `Gmail Trigger` (`ticket-gmail`) filtering for `label:support` unread messages.
   - Add a `Webhook` node (`ticket-webhook`) on path `support-ticket` with header authentication.
   - Add a `Code` node (`normalize-ticket`) to unify payloads, followed by a `Code` node (`mask-pii`) to redact emails, IBANs, card numbers, and phone numbers.
   - Setup classification sub-graph: Connect an OpenRouter chat model (`classifier-model`, `openai/gpt-5.6-luna`) and a Structured Output Parser (`classification-schema`) to an LLM Chain (`classify-ticket`).
   - Add a `Code` node (`assemble-ticket`) and log tickets to PostgreSQL (`log-ticket`).
   - Add a `Switch` node (`policy-router`) routing `auto_answerable`, `drop`, and `restricted` (fallback).
   - For `drop`, add a `Postgres` node (`log-suppressed`).
   - For `auto_answerable`, setup the drafting agent: Connect an OpenRouter model (`drafting-model`, `anthropic/claude-opus-5`), OpenAI embeddings (`answer-embeddings`), a Vector Store PGVector tool (`kb-search-tool`, top K: 6), and a Structured Output Parser (`draft-schema`) to an AI Agent (`draft-answer`).
   - Setup the critic sub-graph: Connect an OpenRouter model (`critic-model`, `openai/gpt-5.6-luna`) and a Structured Output Parser (`critic-schema`) to an LLM Chain (`critic-review`).
   - Add a `Code` node (`merge-verdict`) to consolidate draft and critic outputs.
   - Add an `If` node (`quality-gate`) validating: `critic_overall >= 8`, `critic_verdict == 'send'`, `draft_confidence >= 0.7`, `citation_count >= 1`, and `unsupported_count == 0`.
   - **Auto-Send Branch:** Connect the true branch to a `Code` node (`unmask-answer`) to restore PII, an `If` node (`choose-delivery`) checking channel type, and Gmail nodes (`reply-gmail` or `send-reply`), followed by logging via `log-sent-reply`.
   - **Human Approval Branch:** Connect the false branch of the quality gate to a Gmail `Send and Wait` node (`request-approval`) sending to `SUPPORT_LEAD_EMAIL` with double approval options.
   - Add an `If` node (`approval-granted`) to check if approval was granted. If true, route to `unmask-answer`. If false, route to an escalation packet builder (`build-escalation`).
   - **Escalation Branch:** Connect restricted tickets, declined approvals, and agent errors to `build-escalation`, then to a Gmail node (`escalate-queue`) emailing `SUPPORT_LEAD_EMAIL`, and log via `log-escalation`.
4. **Create Segment C (Weekly Metrics):**
   - Add a `Schedule Trigger` (`metrics-cron`, Mondays at 07:00 AM).
   - Add a `Postgres` node (`metrics-query`) running the metrics aggregation query.
   - Add a `Code` node (`metrics-format`) to build the HTML report.
   - Add a `Gmail` node (`metrics-send`) to email the digest to `SUPPORT_LEAD_EMAIL`.
5. **Credentials Configuration:** Configure valid credentials in n8n for Google Drive OAuth2, Gmail OAuth2, PostgreSQL, OpenAI API, and OpenRouter API.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Support Reply Agent Architecture & Setup Instructions | Provided via workflow sticky notes and README specifications |
| PostgreSQL Schema Requirements | Requires custom schema (`sql/01-schema.sql`) with pgvector extension enabled |
| Environment Placeholders to Replace | `KB_FOLDER_ID` (Google Drive folder) and `SUPPORT_LEAD_EMAIL` (Reviewer / escalation / metrics recipient) |
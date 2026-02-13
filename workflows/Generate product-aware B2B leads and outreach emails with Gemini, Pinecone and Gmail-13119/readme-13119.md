Generate product-aware B2B leads and outreach emails with Gemini, Pinecone and Gmail

https://n8nworkflows.xyz/workflows/generate-product-aware-b2b-leads-and-outreach-emails-with-gemini--pinecone-and-gmail-13119


# Generate product-aware B2B leads and outreach emails with Gemini, Pinecone and Gmail

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

# Generate product-aware B2B leads and outreach emails with Gemini, Pinecone and Gmail

## 1. Workflow Overview

This solution is a multi-stage, end-to-end outbound automation system that:

1) Builds and maintains a **product knowledge base** (RAG) for “Regula” by syncing documents from **Google Drive → Pinecone**, using **Google Sheets** as a change-tracking “record manager”.
2) Uses an AI agent (Gemini + Pinecone tool) to generate **lead search criteria**, then performs **web search (Serper)** to discover relevant companies and stores them in **Postgres**.
3) Enriches those companies by running **AI-driven company research** (Gemini + Serper + Pinecone) and finding **decision makers** via **Hunter.io**, then stores contacts in **Postgres leads**.
4) Generates **personalized outreach emails** via an AI agent and sends them via **Gmail**, then updates lead records to prevent re-sending.

### 1.1 Block A — Knowledge Base Sync (Drive → Pinecone with Sheets-based state)
- Discovers files in a Drive folder, skips folders.
- Downloads each file, hashes content, checks Google Sheet to decide: new / updated / already processed.
- Loads text, splits it, embeds with Gemini embeddings, upserts to Pinecone.
- Writes/updates file hash state in Google Sheets.

### 1.2 Block B — Lead Generation (AI criteria → Serper search → parsing/scoring → Postgres)
- Scheduled run generates search queries using Regula KB (Pinecone retrieval tool).
- Runs each query through Serper, parses results into candidate companies, de-dupes vs existing leads, inserts into `public.search`.

### 1.3 Block C — Contact Generation (Company research + Hunter enrichment → Postgres leads)
- Scheduled run pulls unprocessed companies from `public.search`.
- For each company: waits (throttle), runs research agent (Gemini + Serper tool + Pinecone KB tool), waits (throttle), runs Hunter.
- Parses Hunter results into decision maker contacts.
- If an email exists: inserts into `public.leads`, then marks company as processed (`status=true`).

### 1.4 Block D — Email Outreach (AI email drafting → Gmail → Postgres update)
- Scheduled run pulls unsent leads from `public.leads`.
- For each lead: AI agent drafts subject/body/CTA using KB tool.
- Sends via Gmail and updates lead record with `sent_date` and stored `email_content`.

---

## 2. Block-by-Block Analysis

### Block A — Knowledge Base Sync (Context Creation for AI Agent)

**Overview:**  
Synchronizes documents from a Google Drive folder into Pinecone. Uses SHA256 hashing and a Google Sheet “Documents” tab to avoid reprocessing unchanged files and to trigger re-indexing when a file changes.

**Nodes involved:**
- When clicking ‘Execute workflow’ (Manual Trigger)
- Search files and folders (Google Drive)
- Loop Over Items (Split in Batches)
- nonDownloadableFile (IF)
- Download file (Google Drive)
- createHash (Crypto)
- searchRecordManger (Google Sheets)
- Switch (Switch)
- Add to Record Manger (Google Sheets)
- Update the RecordManger (Google Sheets)
- Download file1 (Google Drive)
- Download file2 (Google Drive)
- Recursive Character Text Splitter (LangChain)
- Default Data Loader (LangChain)
- Embeddings Google Gemini (LangChain)
- Pinecone Vector Store (LangChain Pinecone)
- Pinecone Vector Store1 (LangChain Pinecone)

#### Node details

**1) When clicking ‘Execute workflow’**
- **Type/role:** Manual trigger; entry point for KB sync.
- **Config:** No parameters.
- **Outputs:** → Search files and folders
- **Edge cases:** None.

**2) Search files and folders**
- **Type/role:** Google Drive “fileFolder” list; discovers content in a specific folder.
- **Config choices:**
  - Folder ID points to a “research” folder.
  - Returns all files; requests `mimeType,id,name`.
- **Outputs:** → Loop Over Items
- **Edge cases/failures:** OAuth scope/auth issues, folder not accessible, large folders (execution time).

**3) Loop Over Items**
- **Type/role:** Split in Batches; iterates through Drive items.
- **Config:** Default batching options (batch size not explicitly set).
- **Connections:** Its second output is used for processing items (per connections).
- **Outputs:** → nonDownloadableFile
- **Edge cases:** If batch size default is large, memory/time; if empty, downstream won’t run.

**4) nonDownloadableFile**
- **Type/role:** IF; filters out folders.
- **Condition:** `$json.mimeType == "application/vnd.google-apps.folder"`.
- **True path:** loops back to Loop Over Items (skip folders).
- **False path:** → Download file
- **Edge cases:** Google Docs file types can be “downloadable” but may require export; this workflow uses download operation directly, which can fail for native Google Docs/Sheets/Slides unless Drive node handles export automatically.

**5) Download file**
- **Type/role:** Google Drive download; fetches binary content for hashing.
- **Input:** Current item from nonDownloadableFile false path.
- **Output:** → createHash
- **Edge cases:** Download errors for unsupported types, large files, permission errors.

**6) createHash**
- **Type/role:** Crypto; computes SHA256 on binary data.
- **Config:** `binaryData: true`, output property `hash`.
- **Output:** → searchRecordManger
- **Key data produced:** `$json.hash` plus original metadata.
- **Edge cases:** Missing binary data (download failed) causes hashing failure.

**7) searchRecordManger**
- **Type/role:** Google Sheets lookup; state manager for file ID → previous hash.
- **Config choices:**
  - Document: “Regula Outbound Sales - Emerging Market FinTech Leads”
  - Sheet/tab: “Documents”
  - Filter: `Id == {{$json.id}}`
  - `alwaysOutputData: true` (important for “not found” case)
- **Output:** → Switch
- **Edge cases:** Sheet permissions; lookup returns empty array/object shape differences depending on n8n version; `alwaysOutputData` prevents hard stop but expressions may still reference missing fields.

**8) Switch**
- **Type/role:** Branches by “new / already processed / updated”.
- **Rules (interpreted):**
  - **New Document:** when `$('searchRecordManger').item.json` is empty.
  - **Already processed:** when stored `hashId` equals newly computed `hash`.
  - **Updated Document:** when stored `hashId` differs from new `hash`.
- **Outputs:**
  - Output 0 (“New Document”) → Add to Record Manger
  - Output 1 (“Already processed”) → Loop Over Items (skip)
  - Output 2 (“Updated Document”) → Update the RecordManger
- **Edge cases:** If `searchRecordManger` returns multiple matches, `.item` may not be the intended row; ensure `Id` is unique in the sheet.

**9) Add to Record Manger**
- **Type/role:** Google Sheets appendOrUpdate; creates a tracking row for new files.
- **Config choices:**
  - Matching column: `Id`
  - Writes `Id`, `name`, `hashId` from `createHash` item.
- **Output:** → Download file1
- **Edge cases:** Schema mismatch if sheet columns renamed; append/update requires correct header row.

**10) Update the RecordManger**
- **Type/role:** Google Sheets appendOrUpdate; intended to update hash for changed file.
- **Config:** Matching by `Id`, but **columns.value is empty** in the JSON, meaning it may not actually update hash/name unless set in UI.
- **Output:** → Download file2
- **Edge cases:** This is likely a configuration bug: updated documents may re-index in Pinecone but the sheet hash might not be updated, causing repeated “Updated Document” detection every run.

**11) Download file1**
- **Type/role:** Google Drive download; fetches file again for ingestion (new doc path).
- **Config:** `fileId = $('Loop Over Items').item.json.id`.
- **Output:** → Pinecone Vector Store (as `ai_document` input via Default Data Loader chain)
- **Edge cases:** Same as Download file; duplication of downloads increases cost/time.

**12) Download file2**
- **Type/role:** Google Drive download; ingestion download for updated doc path.
- **Output:** → Pinecone Vector Store1
- **Edge cases:** Same as above.

**13) Recursive Character Text Splitter**
- **Type/role:** LangChain text splitter; chunks extracted text.
- **Config:** `chunkOverlap: 10`. (Sticky note claims chunk size 1000, but chunk size is not shown here; likely default.)
- **Connection:** feeds `ai_textSplitter` → Default Data Loader
- **Edge cases:** Very large docs can produce many chunks; ensure Pinecone limits/billing.

**14) Default Data Loader**
- **Type/role:** LangChain document loader; converts binary file to text documents.
- **Config choices:**
  - Data type: `binary`
  - `splitPages: true`
  - Adds metadata: `file_id`, `file_name` from Loop Over Items item.
- **Connections:** `ai_document` → Pinecone Vector Store and Pinecone Vector Store1
- **Edge cases:** Binary parsing may fail for unsupported mime types (e.g., images). PDF parsing quality varies.

**15) Embeddings Google Gemini**
- **Type/role:** Gemini embeddings model used by Pinecone vector store nodes.
- **Config:** `models/gemini-embedding-001`
- **Connections:** `ai_embedding` → Pinecone Vector Store & Pinecone Vector Store1
- **Edge cases:** API quotas; credentials; model availability by region/project.

**16) Pinecone Vector Store**
- **Type/role:** Upsert embeddings/documents into Pinecone (new doc path).
- **Config:** Mode `insert`, index `n8n-nodes`.
- **Inputs:** documents from Default Data Loader, embeddings from Gemini embeddings.
- **Edge cases:** Namespace not specified; potential mixing of multiple docs unless metadata filtering is used later.

**17) Pinecone Vector Store1**
- **Type/role:** Upsert embeddings/documents into Pinecone (updated doc path).
- **Config:** Mode `insert` with `clearNamespace: true`.
- **Risk:** Clearing namespace can wipe more than intended if namespace strategy isn’t per-file. Ensure namespace is set per document or per corpus in node options (not visible in JSON).
- **Edge cases:** Accidental deletion of unrelated vectors if using a shared namespace.

**Sticky note coverage (context for this block):**
- “Workflow 1 - Context Creation for AI Agent” (content preserved in section 3/5)

---

### Block B — Lead Generation (AI criteria → Serper → parsing → Postgres)

**Overview:**  
Generates ICP/search queries from Regula knowledge base, searches the web for relevant company sites using Serper, filters junk/listicles, de-duplicates against existing leads, and stores candidate companies into a `search` table.

**Nodes involved:**
- Schedule Trigger (lead generation)
- Lead Criteria Agent (LangChain Agent)
- Gemini (LLM)
- KB - Lead Criteria (Pinecone tool)
- Embeddings (Gemini embeddings for tool)
- Parser  (Structured Output Parser)
- Extract Queries (Code)
- Loop Over Queries (Split in Batches)
- Web Search - Serper (HTTP Request)
- Parse Companies (Code)
- Get Existing Leads (Postgres)
- If Leads (IF)
- Extract Lead Names (Code)
- Filter Existing Leads (Code)
- Get Parsed Companies (Code)
- Insert Companies (Postgres)

#### Node details

**1) Schedule Trigger**
- **Type/role:** Time-based entry for lead generation.
- **Config:** Interval rule present but unspecified (empty object).
- **Output:** → Lead Criteria Agent
- **Edge cases:** Misconfigured schedule may run too frequently.

**2) Lead Criteria Agent**
- **Type/role:** LangChain agent that produces structured lead criteria/search queries.
- **Config choices:**
  - Prompt instructs to create search queries targeting **company websites** (not blog content).
  - `hasOutputParser: true` connected to “Parser ”.
  - Uses KB tool (“KB - Lead Criteria”) and Gemini LLM.
- **Outputs:** main → Extract Queries
- **Failure modes:** LLM hallucination; malformed JSON (mitigated by structured parser); missing KB connectivity.

**3) Gemini**
- **Type/role:** Google Gemini chat model for Lead Criteria Agent.
- **Connection:** `ai_languageModel` → Lead Criteria Agent
- **Edge cases:** Quota/model errors.

**4) KB - Lead Criteria**
- **Type/role:** Pinecone vector store as a **tool** for the agent (retrieve-as-tool).
- **Config:** Index `n8n-nodes`, tool description focuses on Regula product/ideal customers.
- **Needs:** Embeddings node to embed queries for retrieval.
- **Edge cases:** If Pinecone empty/not indexed, agent loses grounding.

**5) Embeddings**
- **Type/role:** Gemini embeddings for the retrieval tool.
- **Connection:** `ai_embedding` → KB - Lead Criteria

**6) Parser  **
- **Type/role:** Structured output parser enforcing JSON schema for criteria.
- **Schema example:** industries, keywords, companySize, searchQueries, optional locations.
- **Connection:** `ai_outputParser` → Lead Criteria Agent
- **Edge cases:** If agent output deviates too much, parser fails and block stops.

**7) Extract Queries**
- **Type/role:** Code node; transforms `output.searchQueries` into `{query}` items.
- **Code behavior:** `searchQueries.map(s => ({ query: s }))`
- **Output:** → Loop Over Queries
- **Edge cases:** If `output.searchQueries` missing, throws error.

**8) Loop Over Queries**
- **Type/role:** Iteration over query list.
- **Output (2nd):** → Web Search - Serper
- **Edge cases:** No queries -> nothing happens.

**9) Web Search - Serper**
- **Type/role:** HTTP Request to Serper Google Search API.
- **Config choices:**
  - POST `https://google.serper.dev/search`
  - Body includes `q` with many negative operators to exclude blogs/social/listicles.
  - `num: 20`, `gl: us`, `hl: en`
  - Auth: Header auth credential “Google Serper”
- **Output:** → Parse Companies
- **Failure modes:** 401 (bad key), 429 rate limits, Serper downtime; query operator over-filtering reducing results.

**10) Parse Companies**
- **Type/role:** Code node; filters search results into company candidates and scores them.
- **Key logic:**
  - Excludes URLs containing blog/article/news/forum/etc patterns and many known domains.
  - Excludes titles/snippets matching listicle-like phrases.
  - Extracts company name from domain.
  - Scores likely homepages and “business” URLs (`/solutions`, `/products`, etc.).
  - De-duplicates by domain and returns top 20.
- **Output:** → Get Existing Leads
- **Edge cases:** Legit companies hosted on subdomains or non-standard domains; false negatives due to aggressive filters.

**11) Get Existing Leads**
- **Type/role:** Postgres select from `public.leads` to build a de-duplication set.
- **Config:** `returnAll: true`, `executeOnce: true` (cached once per workflow execution).
- **Output:** → If Leads
- **Edge cases:** Large leads table can slow; schema drift.

**12) If Leads**
- **Type/role:** IF node with “exists id” check.
- **Behavior (as wired):**
  - **True:** → Extract Lead Names
  - **False:** → Get Parsed Companies
- **Important nuance:** This checks whether each incoming item has an `id` field; depending on incoming data shape, may behave unexpectedly. It appears intended to branch depending on whether existing leads were fetched.
- **Edge cases:** This is a fragile design: it relies on item structure rather than explicit “number of leads > 0”.

**13) Extract Lead Names**
- **Type/role:** Code node; builds unique set of existing lead company names.
- **Code expects:** each item has `Company Name` field.
- **Output:** → Filter Existing Leads
- **Edge cases:** Your `leads` table likely uses `company_name` not `"Company Name"`; if so, it will produce an empty list and de-duplication will not work.

**14) Filter Existing Leads**
- **Type/role:** Code node; removes parsed companies whose `name` matches existing lead names (case-insensitive).
- **Inputs:**
  - Uses `$("Parse Companies").all(0)` for new companies.
  - Uses `$input.all(1)[0].json.companies` for blacklist.
- **Output:** → Insert Companies
- **Failure modes:** If second input missing or Extract Lead Names produced unexpected shape, indexing errors.

**15) Get Parsed Companies**
- **Type/role:** Code node; returns all items from Parse Companies (pass-through).
- **Output:** → Insert Companies
- **Purpose:** Fallback path when “no leads” or when IF branch routes here.

**16) Insert Companies**
- **Type/role:** Postgres insert into `public.search`.
- **Config choices:**
  - Maps name, website, snippet, position, score, domain.
  - `onError: continueErrorOutput` (won’t fail whole run on insert errors).
- **Outputs:** back to Loop Over Queries (both outputs are connected to Loop Over Queries, allowing continued looping).
- **Edge cases:** Unique constraint violations (if any), type mismatch for `position` (schema says string), SQL errors.

**Sticky note coverage (context for this block):**
- “Workflow 2 - Lead Generation” (content preserved in section 3/5)

---

### Block C — Contact Generation (Deep research + Hunter enrichment)

**Overview:**  
Processes unhandled companies from `public.search`, produces a structured research summary with an AI agent (grounded by Serper + Pinecone KB), finds decision makers with Hunter.io, and stores verified contacts in `public.leads`. Marks companies as processed.

**Nodes involved:**
- Schedule Trigger1
- Get Companies (Postgres)
- Loop Over Companies (Split in Batches)
- Wait
- Company Research Agent
- Gemini1
- KB - Research (Pinecone tool)
- Embeddings1
- Web Search - Serper1 (HTTP Request Tool)
- Parser (Structured Output Parser)
- Wait Before Email Finder
- Hunter
- Parse Decision Makers (Code)
- If (email exists)
- Insert Lead (Postgres)
- Update Company Status (Postgres)

#### Node details

**1) Schedule Trigger1**
- **Type/role:** Scheduled entry for contact generation.
- **Config:** triggers at hour 1.
- **Output:** → Get Companies

**2) Get Companies**
- **Type/role:** Postgres select from `public.search`.
- **Config choices:** `limit: 5`, where `status = FALSE`.
- **Output:** → Loop Over Companies
- **Edge cases:** If `status` null vs false, records may be skipped; ensure default false.

**3) Loop Over Companies**
- **Type/role:** Processes companies in batches.
- **Output (2nd):** → Wait
- **Edge cases:** If no companies, nothing runs.

**4) Wait**
- **Type/role:** Throttle node.
- **Config:** amount 10 (seconds by default).
- **Output:** → Company Research Agent
- **Edge cases:** Increases runtime; for many companies, workflow may exceed execution limits.

**5) Company Research Agent**
- **Type/role:** LangChain agent that researches a company and how Regula fits.
- **Inputs in prompt:** company name and website from Loop Over Companies.
- **Tools:** 
  - Web Search - Serper1 (as tool)
  - KB - Research (Pinecone tool)
- **Output parser:** “Parser” enforces JSON fields: challenges, complianceNeeds, useCases, recentNews, competitors.
- **Output:** → Wait Before Email Finder
- **Failure modes:** Tool failures (Serper rate limit), parser failures, irrelevant results if company name ambiguous.

**6) Gemini1**
- **Type/role:** Gemini chat model for research agent.
- **Connection:** `ai_languageModel` → Company Research Agent

**7) KB - Research**
- **Type/role:** Pinecone retrieval tool grounding Regula product features.
- **Connection:** `ai_tool` → Company Research Agent
- **Embeddings:** provided by Embeddings1.

**8) Embeddings1**
- **Type/role:** Gemini embeddings to support KB retrieval.

**9) Web Search - Serper1**
- **Type/role:** HTTP Request Tool for agent (tool-enabled).
- **Query:** `{{companyName}} recent news compliance regulatory`, `num:10`.
- **Auth:** Serper header auth.
- **Edge cases:** Tool errors bubble into agent output quality.

**10) Parser**
- **Type/role:** Structured output parser for company research JSON.
- **Edge cases:** Strict schema can fail when agent returns strings vs arrays inconsistently.

**11) Wait Before Email Finder**
- **Type/role:** Throttle before Hunter.
- **Config:** amount 2.

**12) Hunter**
- **Type/role:** Hunter.io domain search.
- **Config:** `domain = Loop Over Companies item.json.domain`, `limit:10`.
- **Output:** → Parse Decision Makers
- **Edge cases:** Hunter plan limits; domain may be wrong/empty; returns generic emails.

**13) Parse Decision Makers**
- **Type/role:** Code node; transforms Hunter results to lead rows.
- **Key logic:**
  - Uses `i.json.value` as email, and `first_name/last_name`, `position`, `linkedin`.
  - Pulls `companyDomain`, `companyName` from Loop Over Companies item.
  - Filters out `"null null"`.
- **Output:** → If
- **Edge cases:** Hunter field names may differ (`email` vs `value`); if so, email extraction fails.

**14) If**
- **Type/role:** Validates email exists (`$json.email` exists).
- **True path:** → Insert Lead
- **False path:** → Update Company Status
- **Edge cases:** Allows any string; doesn’t validate format.

**15) Insert Lead**
- **Type/role:** Postgres insert into `public.leads`.
- **Mapping:** name, email, title, website(companyDomain), linkedin, company_id from Loop Over Companies first item.
- **Note:** `company_research` is not inserted (field exists but removed in mapping). The Email workflow later expects `company_research` in prompt; unless another process fills it, personalization may be missing context.
- **Output:** → Update Company Status
- **Edge cases:** Duplicate emails; missing foreign key company_id if schema expects it.

**16) Update Company Status**
- **Type/role:** Postgres update `public.search` set `status=true` for processed company.
- **Mapping:** `id = Loop Over Companies first json.id`.
- **Output:** → Loop Over Companies (continue next company)
- **Edge cases:** If “first()” doesn’t match current item in batch contexts, wrong row could be updated (rare but possible with certain batching behaviors).

**Sticky note coverage (context for this block):**
- “Workflow 3 - Contact Generation” (content preserved in section 3/5)

---

### Block D — Email Outreach (Personalization → Gmail → Postgres tracking)

**Overview:**  
Selects leads that haven’t been emailed, drafts a tailored email using Gemini grounded by the Regula KB, sends it via Gmail, and records the sent timestamp and content in Postgres.

**Nodes involved:**
- Schedule Trigger2
- Get Leads (Postgres)
- Loop Over Leads (Split in Batches)
- Email Personalization Agent
- Gemini2
- KB - Email (Pinecone tool)
- Embeddings2
- Parser1 (Structured Output Parser)
- Send Gmail
- Update Lead (Postgres)

#### Node details

**1) Schedule Trigger2**
- **Type/role:** Scheduled entry for outreach.
- **Config:** triggers at hour 2.
- **Output:** → Get Leads

**2) Get Leads**
- **Type/role:** Postgres select from `public.leads`.
- **Config:** `limit: 5`, where `sent_date IS NULL`.
- **Output:** → Loop Over Leads
- **Edge cases:** If sent_date stored as non-null string with timezone issues, filter may fail.

**3) Loop Over Leads**
- **Type/role:** Iterates leads.
- **Output (2nd):** → Email Personalization Agent
- **Edge cases:** None.

**4) Email Personalization Agent**
- **Type/role:** LangChain agent to write personalized email (150–200 words).
- **Inputs:** decision maker name/title, company website, and `company_research` from lead record.
- **System message:** Forces JSON output: subject/body(HTML)/cta, with a fixed footer.
- **Tools:** KB - Email for product grounding.
- **Output parser:** Parser1.
- **Output:** → Send Gmail
- **Edge cases:** If `company_research` is null/missing, email becomes generic; parser failures stop sending.

**5) Gemini2**
- **Type/role:** Gemini chat model for email agent.

**6) KB - Email**
- **Type/role:** Pinecone retrieval tool for Regula value props/use cases.
- **Embeddings:** Embeddings2 provides query embeddings.

**7) Embeddings2**
- **Type/role:** Gemini embeddings for KB tool.

**8) Parser1**
- **Type/role:** Structured output parser enforcing `{subject, body, cta}`.
- **Edge cases:** HTML escaping; LLM may return non-JSON; parser fails.

**9) Send Gmail**
- **Type/role:** Sends email via Gmail OAuth2.
- **Config:**
  - To: `Loop Over Leads item.json.email`
  - Subject: `$json.output.subject`
  - Message: `$json.output.body` + newline + `$json.output.cta`
  - `emailType: text` but body is HTML (mismatch): if you want HTML rendering, use HTML email type.
  - `appendAttribution: false`
- **Output:** → Update Lead
- **Failure modes:** Gmail OAuth expiry; daily send limits; invalid recipient.

**10) Update Lead**
- **Type/role:** Postgres update `public.leads` to set sent tracking.
- **Config:**
  - `sent_date = $now`
  - `email_content = JSON.stringify({subject, body, cta: 'Schedule a 15-minute call'})`
  - Uses agent output for subject/body but hardcodes CTA in DB to “Schedule a 15-minute call”.
- **Output:** loops back to Loop Over Leads to process next lead.
- **Edge cases:** `$now` format; email_content column type should accept text/json.

**Sticky note coverage (context for this block):**
- “Workflow 4 - Email Outreach” (content preserved in section 3/5)

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | Manual Trigger | Start KB sync manually | — | Search files and folders | # Workflow 1 - Context Creation for AI Agent / (full note content in section 5) |
| Search files and folders | Google Drive | List files in research folder | When clicking ‘Execute workflow’ | Loop Over Items | Workflow 1 note |
| Loop Over Items | Split in Batches | Iterate over Drive items | Search files and folders / Switch(Already processed) / nonDownloadableFile(true) | nonDownloadableFile | Workflow 1 note |
| nonDownloadableFile | IF | Skip folders, keep files | Loop Over Items | Loop Over Items (true) / Download file (false) | Workflow 1 note |
| Download file | Google Drive | Download file for hashing | nonDownloadableFile(false) | createHash | Workflow 1 note |
| createHash | Crypto | SHA256 hash of file binary | Download file | searchRecordManger | Workflow 1 note |
| searchRecordManger | Google Sheets | Lookup file record by Id | createHash | Switch | Workflow 1 note |
| Switch | Switch | Route New/Updated/Already processed | searchRecordManger | Add to Record Manger / Loop Over Items / Update the RecordManger | Workflow 1 note |
| Add to Record Manger | Google Sheets | Write new file state (Id/name/hash) | Switch(New Document) | Download file1 | Workflow 1 note |
| Update the RecordManger | Google Sheets | Update file state on change | Switch(Updated Document) | Download file2 | Workflow 1 note |
| Download file1 | Google Drive | Download file for ingestion (new) | Add to Record Manger | Pinecone Vector Store | Workflow 1 note |
| Download file2 | Google Drive | Download file for ingestion (updated) | Update the RecordManger | Pinecone Vector Store1 | Workflow 1 note |
| Recursive Character Text Splitter | LangChain Text Splitter | Chunk text for embeddings | (implicit chain) | Default Data Loader | Workflow 1 note |
| Default Data Loader | LangChain Document Loader | Extract text from binary + metadata | Recursive Character Text Splitter | Pinecone Vector Store / Pinecone Vector Store1 | Workflow 1 note |
| Embeddings Google Gemini | Gemini Embeddings | Provide embeddings for Pinecone upsert | — | Pinecone Vector Store / Pinecone Vector Store1 | Workflow 1 note |
| Pinecone Vector Store | Pinecone Vector Store (LangChain) | Upsert vectors (new docs) | Download file1 via loader chain | — | Workflow 1 note |
| Pinecone Vector Store1 | Pinecone Vector Store (LangChain) | Upsert vectors (updated docs; clears namespace) | Download file2 via loader chain | — | Workflow 1 note |
| Schedule Trigger | Schedule Trigger | Start lead generation | — | Lead Criteria Agent | # Workflow 2 - Lead Generation / (full note content in section 5) |
| Lead Criteria Agent | LangChain Agent | Generate ICP + search queries | Schedule Trigger / KB tool | Extract Queries | Workflow 2 note |
| Gemini | Gemini Chat Model | LLM for lead criteria agent | — | Lead Criteria Agent | Workflow 2 note |
| KB - Lead Criteria | Pinecone Vector Store Tool | Product KB retrieval tool (criteria stage) | Embeddings | Lead Criteria Agent | Workflow 2 note |
| Embeddings | Gemini Embeddings | Embeddings for KB tool | — | KB - Lead Criteria | Workflow 2 note |
| Parser  | Structured Output Parser | Enforce criteria JSON schema | — | Lead Criteria Agent | Workflow 2 note |
| Extract Queries | Code | Convert searchQueries into items | Lead Criteria Agent | Loop Over Queries | Workflow 2 note |
| Loop Over Queries | Split in Batches | Iterate search queries | Extract Queries / Insert Companies | Web Search - Serper | Workflow 2 note |
| Web Search - Serper | HTTP Request | Serper Google search | Loop Over Queries | Parse Companies | Workflow 2 note |
| Parse Companies | Code | Filter/score/unique company sites | Web Search - Serper | Get Existing Leads | Workflow 2 note |
| Get Existing Leads | Postgres | Load existing leads for de-duplication | Parse Companies | If Leads | Workflow 2 note |
| If Leads | IF | Branch depending on item having id | Get Existing Leads | Extract Lead Names / Get Parsed Companies | Workflow 2 note |
| Extract Lead Names | Code | Build set of existing company names | If Leads(true) | Filter Existing Leads | Workflow 2 note |
| Filter Existing Leads | Code | Remove duplicates from parsed companies | Extract Lead Names (+ Parse Companies via expression) | Insert Companies | Workflow 2 note |
| Get Parsed Companies | Code | Pass-through parsed companies | If Leads(false) | Insert Companies | Workflow 2 note |
| Insert Companies | Postgres | Insert candidates into `public.search` | Filter Existing Leads / Get Parsed Companies | Loop Over Queries | Workflow 2 note |
| Schedule Trigger1 | Schedule Trigger | Start contact generation | — | Get Companies | # Workflow 3 - Contact Generation / (full note content in section 5) |
| Get Companies | Postgres | Select unprocessed companies | Schedule Trigger1 | Loop Over Companies | Workflow 3 note |
| Loop Over Companies | Split in Batches | Iterate companies | Get Companies / Update Company Status | Wait | Workflow 3 note |
| Wait | Wait | Throttle before research | Loop Over Companies | Company Research Agent | Workflow 3 note |
| Company Research Agent | LangChain Agent | Deep research summary JSON | Wait / KB tool / Serper tool | Wait Before Email Finder | Workflow 3 note |
| Gemini1 | Gemini Chat Model | LLM for research agent | — | Company Research Agent | Workflow 3 note |
| KB - Research | Pinecone Vector Store Tool | Product KB retrieval tool (research stage) | Embeddings1 | Company Research Agent | Workflow 3 note |
| Embeddings1 | Gemini Embeddings | Embeddings for KB tool | — | KB - Research | Workflow 3 note |
| Web Search - Serper1 | HTTP Request Tool | Agent tool for web search | — | Company Research Agent | Workflow 3 note |
| Parser | Structured Output Parser | Enforce research JSON schema | — | Company Research Agent | Workflow 3 note |
| Wait Before Email Finder | Wait | Throttle before Hunter | Company Research Agent | Hunter | Workflow 3 note |
| Hunter | Hunter | Find emails/people by domain | Wait Before Email Finder | Parse Decision Makers | Workflow 3 note |
| Parse Decision Makers | Code | Normalize Hunter results into leads | Hunter | If | Workflow 3 note |
| If | IF | Check email exists | Parse Decision Makers | Insert Lead (true) / Update Company Status (false) | Workflow 3 note |
| Insert Lead | Postgres | Insert decision maker into `public.leads` | If(true) | Update Company Status | Workflow 3 note |
| Update Company Status | Postgres | Mark company `status=true` | Insert Lead / If(false) | Loop Over Companies | Workflow 3 note |
| Schedule Trigger2 | Schedule Trigger | Start outreach emails | — | Get Leads | # Workflow 4 - Email Outreach / (full note content in section 5) |
| Get Leads | Postgres | Select unsent leads | Schedule Trigger2 | Loop Over Leads | Workflow 4 note |
| Loop Over Leads | Split in Batches | Iterate leads to email | Get Leads / Update Lead | Email Personalization Agent | Workflow 4 note |
| Email Personalization Agent | LangChain Agent | Draft personalized email JSON | Loop Over Leads / KB tool | Send Gmail | Workflow 4 note |
| Gemini2 | Gemini Chat Model | LLM for email agent | — | Email Personalization Agent | Workflow 4 note |
| KB - Email | Pinecone Vector Store Tool | Product KB tool (email stage) | Embeddings2 | Email Personalization Agent | Workflow 4 note |
| Embeddings2 | Gemini Embeddings | Embeddings for KB tool | — | KB - Email | Workflow 4 note |
| Parser1 | Structured Output Parser | Enforce email JSON schema | — | Email Personalization Agent | Workflow 4 note |
| Send Gmail | Gmail | Send outreach email | Email Personalization Agent | Update Lead | Workflow 4 note |
| Update Lead | Postgres | Save sent_date + email_content | Send Gmail | Loop Over Leads | Workflow 4 note |
| Manual Trigger / Schedule Trigger nodes not connected together | — | Multiple entry points | — | — | (No additional sticky note beyond block notes) |

---

## 4. Reproducing the Workflow from Scratch

### Prerequisites (services & credentials)
1. **Google Drive OAuth2** credential (read access to the folder containing KB docs).
2. **Google Sheets OAuth2** credential (read/write to the sheet with the “Documents” tab).
3. **Pinecone** credential (API key) and a created index (here: `n8n-nodes`).
4. **Gemini / Google PaLM (Gemini)** credential for:
   - Chat model nodes
   - Embeddings model `models/gemini-embedding-001`
5. **Serper.dev** API key stored as **HTTP Header Auth** credential:
   - Header typically `X-API-KEY: <key>` (ensure your credential matches Serper requirements).
6. **Postgres** credential with tables:
   - `public.search` (company candidates)
   - `public.leads` (decision makers + email status)
7. **Hunter.io** API credential.
8. **Gmail OAuth2** credential for sending.

---

### A) Build Block A — KB Sync (Manual run)

1) Create **Manual Trigger** node named **“When clicking ‘Execute workflow’”**.

2) Add **Google Drive** node named **“Search files and folders”**:
   - Resource: *File/Folder*
   - Operation: *Search* (or list/search equivalent)
   - Folder ID: your KB folder
   - Return All: true
   - Fields: `mimeType`, `id`, `name`

3) Add **Split in Batches** named **“Loop Over Items”** and connect:
   - Manual Trigger → Search files and folders → Loop Over Items

4) Add **IF** node named **“nonDownloadableFile”**:
   - Condition: `{{$json.mimeType}} equals application/vnd.google-apps.folder`
   - True output: connect back to **Loop Over Items** (to skip folder items)
   - False output: continue processing

5) Add **Google Drive Download** node named **“Download file”**:
   - Operation: Download
   - File ID: `={{$json.id}}`
   - Connect nonDownloadableFile (false) → Download file

6) Add **Crypto** node named **“createHash”**:
   - Operation/Type: SHA256
   - Binary data: true
   - Output property: `hash`
   - Connect Download file → createHash

7) Add **Google Sheets** node named **“searchRecordManger”**:
   - Operation: Read/Lookup (filter rows)
   - Document: your sheet
   - Sheet/tab: `Documents`
   - Filter: `Id` equals `={{$json.id}}`
   - Enable **Always Output Data**
   - Connect createHash → searchRecordManger

8) Add **Switch** node named **“Switch”** with 3 outputs:
   - Rule 1 (“New Document”): detect empty lookup result (use “is empty” style condition).
   - Rule 2 (“Already processed”): `stored hashId == new hash`
   - Rule 3 (“Updated Document”): `stored hashId != new hash`
   - Connect searchRecordManger → Switch

9) Add **Google Sheets appendOrUpdate** node named **“Add to Record Manger”**:
   - Document: your sheet
   - Sheet: `Documents`
   - Operation: Append or Update
   - Matching column: `Id`
   - Map values:
     - Id: `={{$('createHash').item.json.id}}`
     - name: `={{$('createHash').item.json.name}}`
     - hashId: `={{$('createHash').item.json.hash}}`
   - Connect Switch(New Document) → Add to Record Manger

10) Add **Google Sheets appendOrUpdate** node named **“Update the RecordManger”**:
   - Same sheet and matching column `Id`
   - **Important:** set mapped values at least for:
     - hashId: new hash
     - name: file name
   - Connect Switch(Updated Document) → Update the RecordManger

11) Add **Google Drive Download** node named **“Download file1”**:
   - File ID: `={{$('Loop Over Items').item.json.id}}`
   - Connect Add to Record Manger → Download file1

12) Add **Google Drive Download** node named **“Download file2”**:
   - File ID: `={{$('Loop Over Items').item.json.id}}`
   - Connect Update the RecordManger → Download file2

13) Add **Recursive Character Text Splitter**:
   - Set chunk overlap to 10
   - (Optionally set chunk size to 1000 to match the sticky note intent.)

14) Add **Default Data Loader**:
   - Data type: Binary
   - Split pages: true
   - Metadata:
     - `file_id = {{$('Loop Over Items').item.json.id}}`
     - `file_name = {{$('Loop Over Items').item.json.name}}`

15) Add **Gemini Embeddings** node named **“Embeddings Google Gemini”**:
   - Model: `models/gemini-embedding-001`

16) Add **Pinecone Vector Store** node named **“Pinecone Vector Store”**:
   - Mode: Insert
   - Index: your Pinecone index
   - Connect:
     - Text Splitter → Default Data Loader (as AI chain)
     - Default Data Loader → Pinecone Vector Store (AI document)
     - Embeddings Google Gemini → Pinecone Vector Store (AI embedding)
   - Connect Download file1 into the ingestion chain (binary → loader).

17) Add second **Pinecone Vector Store** node named **“Pinecone Vector Store1”** for updated docs:
   - Mode: Insert
   - Option: Clear Namespace = true (only if you have a safe namespace strategy)
   - Connect Default Data Loader + Embeddings to it as well
   - Connect Download file2 into the same ingestion chain.

18) Connect Switch(Already processed) → Loop Over Items to continue.

---

### B) Build Block B — Lead Generation

19) Add **Schedule Trigger** named **“Schedule Trigger”** (configure frequency).

20) Add **Gemini Chat Model** node named **“Gemini”**.

21) Add **Pinecone Vector Store** tool node named **“KB - Lead Criteria”**:
   - Mode: Retrieve as tool
   - Index: same Pinecone index
   - Tool description: Regula product/ideal customers

22) Add **Gemini Embeddings** named **“Embeddings”**:
   - Connect as embedding source to KB - Lead Criteria.

23) Add **Structured Output Parser** named **“Parser ”**:
   - Provide a JSON schema example including `industries`, `keywords`, `companySize`, `searchQueries`.

24) Add **LangChain Agent** named **“Lead Criteria Agent”**:
   - Prompt: generate lead search criteria JSON (as in workflow).
   - Enable output parser and connect Parser .
   - Connect tools: KB - Lead Criteria.
   - Connect model: Gemini.
   - Connect Schedule Trigger → Lead Criteria Agent.

25) Add **Code** node **“Extract Queries”** to map `output.searchQueries` into `{query}` items.
   - Connect Lead Criteria Agent → Extract Queries.

26) Add **Split in Batches** node **“Loop Over Queries”**.
   - Connect Extract Queries → Loop Over Queries.

27) Add **HTTP Request** node **“Web Search - Serper”**:
   - POST `https://google.serper.dev/search`
   - Header `Content-Type: application/json`
   - Auth via HTTP Header Auth credential
   - JSON body with `q` using negative operators, `num:20`, `gl:us`, `hl:en`.
   - Connect Loop Over Queries → Web Search - Serper.

28) Add **Code** node **“Parse Companies”**:
   - Paste filtering/scoring code (adapt exclusions as needed).
   - Connect Web Search - Serper → Parse Companies.

29) Add **Postgres** node **“Get Existing Leads”**:
   - Operation: Select
   - Table: `public.leads`
   - Return all (or at least company names)
   - Execute once: true
   - Connect Parse Companies → Get Existing Leads.

30) Add **IF** node **“If Leads”**:
   - Preferably change logic to something explicit (e.g., count > 0). If reproducing exactly: “id exists”.
   - Connect Get Existing Leads → If Leads.

31) Add **Code** node **“Extract Lead Names”**:
   - Ensure it reads the correct column from your leads table (likely `company_name`).
   - Connect If Leads (true) → Extract Lead Names.

32) Add **Code** node **“Filter Existing Leads”**:
   - Input 0: parsed companies
   - Input 1: extracted lead names
   - Connect Extract Lead Names → Filter Existing Leads.
   - Also ensure Parse Companies is accessible (or wire Parse Companies directly as input 0).

33) Add **Code** node **“Get Parsed Companies”** for fallback:
   - Connect If Leads (false) → Get Parsed Companies.

34) Add **Postgres** node **“Insert Companies”**:
   - Insert into `public.search` fields (name, website, snippet, position, score, domain)
   - Set node “On Error: Continue”
   - Connect Filter Existing Leads → Insert Companies
   - Connect Get Parsed Companies → Insert Companies
   - Connect Insert Companies → Loop Over Queries to continue loop.

---

### C) Build Block C — Contact Generation

35) Add **Schedule Trigger** named **“Schedule Trigger1”** (e.g., 1 AM).

36) Add **Postgres** node **“Get Companies”**:
   - Select from `public.search`
   - Where `status = FALSE`
   - Limit 5
   - Connect Schedule Trigger1 → Get Companies → Loop Over Companies (Split in Batches).

37) Add **Wait** node **“Wait”** (10 seconds).
   - Connect Loop Over Companies → Wait.

38) Add **Gemini Chat** node **“Gemini1”**.

39) Add **Serper HTTP Request Tool** node **“Web Search - Serper1”**:
   - Tool description: Google Serper API
   - Query body: `{{companyName}} recent news compliance regulatory`
   - Auth header as before.

40) Add **Pinecone Tool** node **“KB - Research”** + **Embeddings1** (Gemini embeddings).

41) Add **Structured Output Parser** node **“Parser”** with research schema.

42) Add **LangChain Agent** node **“Company Research Agent”**:
   - Provide research prompt and system message.
   - Attach tools: Web Search - Serper1, KB - Research.
   - Attach model: Gemini1
   - Attach output parser: Parser
   - Connect Wait → Company Research Agent.

43) Add **Wait** node **“Wait Before Email Finder”** (2 seconds).
   - Connect Company Research Agent → Wait Before Email Finder.

44) Add **Hunter** node:
   - Domain: `={{$('Loop Over Companies').item.json.domain}}`
   - Limit: 10
   - Connect Wait Before Email Finder → Hunter.

45) Add **Code** node **“Parse Decision Makers”**:
   - Normalize Hunter result fields to `{name,email,title,linkedin,companyDomain,companyName}`.
   - Connect Hunter → Parse Decision Makers.

46) Add **IF** node **“If”**:
   - Condition: email exists
   - Connect Parse Decision Makers → If.

47) Add **Postgres** node **“Insert Lead”**:
   - Insert into `public.leads` at least: name,email,title,linkedin,website,company_id
   - (Recommended) also store `company_research` output from research agent in the lead record, because the email agent expects it.
   - Connect If(true) → Insert Lead.

48) Add **Postgres** node **“Update Company Status”**:
   - Update `public.search` set `status = true` for company id
   - Connect:
     - Insert Lead → Update Company Status
     - If(false) → Update Company Status
   - Connect Update Company Status → Loop Over Companies to continue.

---

### D) Build Block D — Email Outreach

49) Add **Schedule Trigger** named **“Schedule Trigger2”** (e.g., 2 AM).

50) Add **Postgres** node **“Get Leads”**:
   - Select from `public.leads`
   - Where `sent_date IS NULL`
   - Limit 5
   - Connect Schedule Trigger2 → Get Leads → Loop Over Leads (Split in Batches).

51) Add **Gemini Chat** node **“Gemini2”**.

52) Add **Pinecone Tool** node **“KB - Email”** + **Embeddings2**.

53) Add **Structured Output Parser** node **“Parser1”** enforcing `{subject, body, cta}`.

54) Add **LangChain Agent** node **“Email Personalization Agent”**:
   - Prompt uses lead fields (name, title, website, company_research).
   - Tools: KB - Email
   - Model: Gemini2
   - Parser: Parser1
   - Connect Loop Over Leads → Email Personalization Agent.

55) Add **Gmail** node **“Send Gmail”**:
   - To: `={{$('Loop Over Leads').item.json.email}}`
   - Subject: `={{$json.output.subject}}`
   - Body: `={{$json.output.body}}` (set email type to HTML if you want HTML rendering)
   - Connect Email Personalization Agent → Send Gmail.

56) Add **Postgres** node **“Update Lead”**:
   - Update `public.leads` by `id`
   - Set `sent_date = {{$now}}`
   - Set `email_content` to a serialized JSON of subject/body/cta
   - Connect Send Gmail → Update Lead → Loop Over Leads.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| **Workflow 1 - Context Creation for AI Agent**: RAG pipeline syncing Drive docs to Pinecone with Sheets record manager; hashing-based change detection; ingestion via loader/splitter/embeddings; Pinecone upsert; updated docs clear namespace. | Sticky note “Workflow 1 - Context Creation for AI Agent” |
| **Workflow 2 - Lead Generation**: AI generates search criteria from Regula KB; Serper web search with negative operators; JS parsing filters listicles/blogs and scores likely company sites; Postgres de-duplication; insert into `search`. | Sticky note “Workflow 2 - Lead Generation” |
| **Workflow 3 - Contact Generation**: Pull unprocessed companies; throttle; research agent uses Serper + Pinecone; Hunter finds emails; parse decision makers; insert into leads; mark company processed. | Sticky note “Workflow 3 - Contact Generation” |
| **Workflow 4 - Email Outreach**: Pull unsent leads; AI drafts email using KB; send via Gmail; update DB with sent_date and email_content for tracking. | Sticky note “Workflow 4 - Email Outreach” |

### Notable implementation risks to address before production
- **Update the RecordManger node maps no values** → hashes may never update, causing repeated reprocessing.
- **Pinecone clearNamespace** is dangerous unless namespaces are per-document/per-corpus; otherwise it can wipe unrelated vectors.
- **Lead de-duplication likely broken** because Extract Lead Names expects `"Company Name"` which may not exist in Postgres output.
- **Email agent expects `company_research`** but Insert Lead does not store it; personalization may be low-context.
- **Gmail node set to “text” while body is HTML**; switch to HTML to render properly.
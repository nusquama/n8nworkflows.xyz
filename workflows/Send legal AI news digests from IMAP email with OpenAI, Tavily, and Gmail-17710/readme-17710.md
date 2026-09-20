Send legal AI news digests from IMAP email with OpenAI, Tavily, and Gmail

https://n8nworkflows.xyz/workflows/send-legal-ai-news-digests-from-imap-email-with-openai--tavily--and-gmail-17710


# Send legal AI news digests from IMAP email with OpenAI, Tavily, and Gmail

### 1. Workflow Overview

This workflow automates the collection, filtering, and deep-dive research of legal industry AI news from an IMAP email inbox. It regularly polls an email account, scores unread messages for relevance regarding legal artificial intelligence, purges irrelevant or processed emails, aggregates valid insights, enhances them via web search, and compiles a clean, branded HTML email digest dispatched via Gmail.

The architecture is grouped into three distinct logical blocks:
- **1.1 Trigger & Extract:** Periodically activates, connects to an IMAP mailbox, retrieves unread messages, and normalizes email payload fields.
- **1.2 AI Relevance Filter:** Evaluates extracted emails through an OpenAI agent, parses the resulting classification, filters out irrelevant entries, and cleans up the mailbox by deleting processed messages.
- **1.3 Research & Send:** Aggregates verified topics, delegates deep research to an agent equipped with a Tavily web search tool, transforms the markdown findings into structured HTML, and emails the digest.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Trigger & Extract
- **Overview:** Sets a routine execution cadence, retrieves unread messages from an IMAP server, and isolates the necessary text and metadata fields for downstream analysis.
- **Nodes Involved:** 
  - `Schedule — Every 5 Hours`
  - `IMAP — Fetch Unread Emails`
  - `Set — Extract Email Fields`
- **Node Details:**
  - **Schedule — Every 5 Hours**
    - *Type & Role:* `n8n-nodes-base.scheduleTrigger` (Trigger)
    - *Configuration:* Executes every 5 hours.
    - *Expressions:* None.
    - *Connections:* Output connects to `IMAP — Fetch Unread Emails`.
    - *Edge Cases:* Server downtime or overlapping executions if interval is set too aggressively.
  - **IMAP — Fetch Unread Emails**
    - *Type & Role:* `n8n-nodes-imap.imap` (Community Input Node)
    - *Configuration:* Fetches a limit of 10 unread messages (`seen: false`) from the `INBOX` path.
    - *Credentials:* Requires core IMAP account authentication.
    - *Expressions:* None.
    - *Connections:* Input from `Schedule — Every 5 Hours`, output to `Set — Extract Email Fields`.
    - *Edge Cases:* Authentication timeouts, network drops, or mailbox path mismatches.
  - **Set — Extract Email Fields**
    - *Type & Role:* `n8n-nodes-base.set` (Data Transformation)
    - *Configuration:* Assigns specific paths to simplified variable names.
    - *Expressions:* 
      - `emailUid`: `={{ $json.uid || $json.id || $json.messageId }}`
      - `emailSubject`: `={{ $json.envelope.subject }}`
      - `emailFrom`: `={{ $json.envelope.from[0].name }}`
      - `emailBody`: `={{ $json.htmlContent }}`
      - `emailDate`: `={{ $json.envelope.date }}`
    - *Connections:* Input from `IMAP`, output to `OpenAI Agent — Score Relevance`.
    - *Edge Cases:* Missing envelope properties throwing reference errors if email formatting is unusual.

---

#### Block 1.2: AI Relevance Filter
- **Overview:** Submits email content to an LLM agent to score its contextual relevance to legal AI, parses the raw string response into structured JSON, filters matching items, and purges processed emails from the IMAP server.
- **Nodes Involved:**
  - `OpenAI Agent — Score Relevance`
  - `OpenAI — Relevance Model`
  - `Code — Parse AI Analysis`
  - `Filter — Keep Relevant Only`
  - `IMAP — Delete Processed Email`
- **Node Details:**
  - **OpenAI Agent — Score Relevance**
    - *Type & Role:* `@n8n/n8n-nodes-langchain.agent` (AI Processing)
    - *Configuration:* Analyzes email subject, sender, date, and body against explicit legal AI criteria and demands strict JSON responses.
    - *Expressions:* Dynamic prompt referencing `emailSubject`, `emailFrom`, `emailDate`, and `emailBody`.
    - *Connections:* Input from `Set — Extract Email Fields`; connected to `OpenAI — Relevance Model` (AI language model link); output to `Code — Parse AI Analysis`.
    - *Edge Cases:* Rate-limit throttling or unexpected unstructured model outputs.
  - **OpenAI — Relevance Model**
    - *Type & Role:* `@n8n/n8n-nodes-langchain.lmChatOpenAi` (Language Model Sub-Node)
    - *Configuration:* Uses `gpt-4o-mini`.
    - *Credentials:* Requires OpenAI API credentials.
  - **Code — Parse AI Analysis**
    - *Type & Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration:* Attempts direct JSON parsing of the agent output, falls back to regex matching, and injects boolean flags (`isRelevant`).
    - *Expressions:* Processes items from `$input.all()`.
    - *Connections:* Input from Agent; dual outputs to `Filter — Keep Relevant Only` and `IMAP — Delete Processed Email`.
    - *Edge Cases:* Completely malformed string returns causing fallback logic to execute.
  - **Filter — Keep Relevant Only**
    - *Type & Role:* `n8n-nodes-base.filter` (Conditional Router)
    - *Configuration:* Keeps items where `isRelevant` strictly equals `true`.
    - *Expressions:* `={{ $json.isRelevant }}` equals `true`.
    - *Connections:* Input from Code parser, output to `Aggregate — Collect Topics`.
  - **IMAP — Delete Processed Email**
    - *Type & Role:* `n8n-nodes-imap.imap` (Cleanup Action)
    - *Configuration:* Deletes messages based on UID.
    - *Credentials:* Core IMAP credentials.
    - *Expressions:* `={{ $('IMAP — Fetch Unread Emails').item.json.uid }}`
    - *Connections:* Input from Code parser.
    - *Edge Cases:* UID mismatch or permission errors preventing deletion.

---

#### Block 1.3: Research & Send
- **Overview:** Aggregates surviving topics, verifies conditions, triggers an advanced research agent with web-search capabilities, converts markdown findings into a designed HTML layout, and dispatches the digest through Gmail.
- **Nodes Involved:**
  - `Aggregate — Collect Topics`
  - `IF — Has Topics`
  - `OpenAI Agent — Research Topics`
  - `OpenAI — Research Model`
  - `Tavily — Web Search`
  - `Code — Build HTML Email`
  - `Gmail — Send Digest`
- **Node Details:**
  - **Aggregate — Collect Topics**
    - *Type & Role:* `n8n-nodes-base.aggregate` (Data Collection)
    - *Configuration:* Aggregates all item data into a single array payload.
    - *Connections:* Input from Filter, output to `IF — Has Topics`.
  - **IF — Has Topics**
    - *Type & Role:* `n8n-nodes-base.if` (Branching Logic)
    - *Configuration:* Evaluates whether the aggregated array length is greater than zero.
    - *Expressions:* `={{ $json.data.length }}` > `0`.
    - *Connections:* Input from Aggregate, true-branch output to `OpenAI Agent — Research Topics`.
  - **OpenAI Agent — Research Topics**
    - *Type & Role:* `@n8n/n8n-nodes-langchain.agent` (AI Processing with Tools)
    - *Configuration:* Acts as a legal technology research assistant, structuring deep-dive reports with markdown.
    - *Expressions:* Serialized JSON string of primary topics and summaries.
    - *Connections:* Input from IF node; connected to `OpenAI — Research Model` and `Tavily — Web Search`; output to `Code — Build HTML Email`.
  - **OpenAI — Research Model**
    - *Type & Role:* `@n8n/n8n-nodes-langchain.lmChatOpenAi` (Language Model Sub-Node)
    - *Configuration:* Uses `gpt-4o`.
    - *Credentials:* OpenAI API credentials.
  - **Tavily — Web Search**
    - *Type & Role:* `@tavily/n8n-nodes-tavily.tavilyTool` (AI Tool Sub-Node)
    - *Configuration:* Executes web lookups to gather current intelligence.
    - *Expressions:* `={{ $json.data[0].analysis.primaryTopic }}`
    - *Credentials:* Tavily API credentials.
  - **Code — Build HTML Email**
    - *Type & Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration:* Converts Markdown elements to styled HTML components and wraps them into a responsive email template.
    - *Expressions:* Reads `$input.first().json.output`.
    - *Connections:* Input from Research Agent, output to `Gmail — Send Digest`.
  - **Gmail — Send Digest**
    - *Type & Role:* `n8n-nodes-base.gmail` (Email Dispatch)
    - *Configuration:* Sends email messages with attribution disabled.
    - *Credentials:* Gmail OAuth2 credentials.
    - *Expressions:* 
      - Send To: Hardcoded recipient (e.g., `user@example.com`)
      - Subject: `={{ $json.subject }}`
      - Message: `={{ $json.htmlEmail }}`
    - *Connections:* Input from HTML builder node.
    - *Edge Cases:* OAuth token expiration or invalid recipient format.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Overview | `n8n-nodes-base.stickyNote` | Documentation | None | None | Scan email folder and send AI digest for legal news<br>Monitors an IMAP mailbox, uses AI to keep only emails about AI in the legal industry, researches each relevant topic with web search, and emails a branded HTML digest.<br><br>### How it works<br>1. **Schedule — Every 5 Hours** polls the inbox on a timer.<br>2. **IMAP — Fetch Unread Emails** retrieves unread messages; **Set — Extract Email Fields** pulls subject, sender, date, body, UID.<br>3. **OpenAI Agent — Score Relevance** classifies each email as legal-AI relevant or not (JSON), and the source email is deleted from the inbox.<br>4. **Filter — Keep Relevant Only** and **Aggregate — Collect Topics** gather the matches; **IF — Has Topics** continues only when at least one is found.<br>5. **OpenAI Agent — Research Topics** expands each topic using **Tavily — Web Search** and returns a Markdown report.<br>6. **Code — Build HTML Email** converts the report to branded HTML and **Gmail — Send Digest** delivers it.<br><br>### Setup<br>1. Install the community nodes `n8n-nodes-imap` and `@tavily/n8n-nodes-tavily` (Settings → Community Nodes). Self-hosted n8n only.<br>2. Add credentials: IMAP, OpenAI, Tavily, Gmail OAuth2.<br>3. Set the recipient in the **Gmail — Send Digest** node.<br><br>### Customization<br>- Change the polling interval in the Schedule node.<br>- Edit the topic list in **OpenAI Agent — Score Relevance** to target a different domain.<br>- Point the IMAP node at a specific subfolder.<br><br>Support: support@legalgpts.com<br>https://automatedintelligentsolutions.com |
| Community Node Warning | `n8n-nodes-base.stickyNote` | Warning | None | None | ⚠️ Community Nodes Required<br>Install both (self-hosted n8n only):<br><br>**n8n-nodes-imap**<br>**@tavily/n8n-nodes-tavily**<br><br>Settings → Community Nodes → Install |
| Section: Trigger & Extract | `n8n-nodes-base.stickyNote` | Section Marker | None | None | Trigger & Extract<br>Poll inbox on a timer; extract subject, sender, date, body, and UID. |
| Section: AI Relevance Filter | `n8n-nodes-base.stickyNote` | Section Marker | None | None | AI Relevance Filter<br>Score each email for legal-AI relevance; drop non-matches and delete the source email. |
| Section: Research & Send | `n8n-nodes-base.stickyNote` | Section Marker | None | None | Research & Send<br>Research each topic via web search, build a branded HTML email, and send the digest. |
| Schedule — Every 5 Hours | `n8n-nodes-base.scheduleTrigger` | Trigger | None | IMAP — Fetch Unread Emails | Trigger & Extract<br>Poll inbox on a timer; extract subject, sender, date, body, and UID. |
| IMAP — Fetch Unread Emails | `n8n-nodes-imap.imap` | Email Retrieval | Schedule — Every 5 Hours | Set — Extract Email Fields | Trigger & Extract<br>Poll inbox on a timer; extract subject, sender, date, body, and UID. |
| Set — Extract Email Fields | `n8n-nodes-base.set` | Data Transformation | IMAP — Fetch Unread Emails | OpenAI Agent — Score Relevance | Trigger & Extract<br>Poll inbox on a timer; extract subject, sender, date, body, and UID. |
| OpenAI Agent — Score Relevance | `@n8n/n8n-nodes-langchain.agent` | AI Classification | Set — Extract Email Fields | Code — Parse AI Analysis | AI Relevance Filter<br>Score each email for legal-AI relevance; drop non-matches and delete the source email. |
| OpenAI — Relevance Model | `@n8n/n8n-nodes-langchain.lmChatOpenAi` | LLM Provider | None | OpenAI Agent — Score Relevance | AI Relevance Filter<br>Score each email for legal-AI relevance; drop non-matches and delete the source email. |
| Code — Parse AI Analysis | `n8n-nodes-base.code` | Data Formatting | OpenAI Agent — Score Relevance | Filter — Keep Relevant Only, IMAP — Delete Processed Email | AI Relevance Filter<br>Score each email for legal-AI relevance; drop non-matches and delete the source email. |
| Filter — Keep Relevant Only | `n8n-nodes-base.filter` | Conditional Filtering | Code — Parse AI Analysis | Aggregate — Collect Topics | AI Relevance Filter<br>Score each email for legal-AI relevance; drop non-matches and delete the source email. |
| IMAP — Delete Processed Email | `n8n-nodes-imap.imap` | Email Deletion | Code — Parse AI Analysis | None | AI Relevance Filter<br>Score each email for legal-AI relevance; drop non-matches and delete the source email. |
| Aggregate — Collect Topics | `n8n-nodes-base.aggregate` | Data Aggregation | Filter — Keep Relevant Only | IF — Has Topics | Research & Send<br>Research each topic via web search, build a branded HTML email, and send the digest. |
| IF — Has Topics | `n8n-nodes-base.if` | Condition Branching | Aggregate — Collect Topics | OpenAI Agent — Research Topics | Research & Send<br>Research each topic via web search, build a branded HTML email, and send the digest. |
| OpenAI Agent — Research Topics | `@n8n/n8n-nodes-langchain.agent` | Research Agent | IF — Has Topics | Code — Build HTML Email | Research & Send<br>Research each topic via web search, build a branded HTML email, and send the digest. |
| OpenAI — Research Model | `@n8n/n8n-nodes-langchain.lmChatOpenAi` | LLM Provider | None | OpenAI Agent — Research Topics | Research & Send<br>Research each topic via web search, build a branded HTML email, and send the digest. |
| Tavily — Web Search | `@tavily/n8n-nodes-tavily.tavilyTool` | Web Search Tool | None | OpenAI Agent — Research Topics | Research & Send<br>Research each topic via web search, build a branded HTML email, and send the digest. |
| Code — Build HTML Email | `n8n-nodes-base.code` | HTML Template Generation | OpenAI Agent — Research Topics | Gmail — Send Digest | Research & Send<br>Research each topic via web search, build a branded HTML email, and send the digest. |
| Gmail — Send Digest | `n8n-nodes-base.gmail` | Email Dispatch | Code — Build HTML Email | None | Research & Send<br>Research each topic via web search, build a branded HTML email, and send the digest. |

---

### 4. Reproducing the Workflow from Scratch

Follow these manual steps to rebuild the entire architecture inside a self-hosted n8n environment:

1. **Prerequisites & Community Nodes:**
   - Ensure your instance is self-hosted.
   - Go to **Settings → Community Nodes**, select **Install**, and install `n8n-nodes-imap` and `@tavily/n8n-nodes-tavily`.

2. **Node Creation & Configuration:**
   - **Step 1:** Create a **Schedule Trigger** node. Set interval to every 5 hours.
   - **Step 2:** Create an **IMAP** node (`IMAP — Fetch Unread Emails`). Set resource to `email`, action to `Get Many`, mark flags unread (`seen: false`), and target mailbox path `INBOX`. Configure core IMAP credentials.
   - **Step 3:** Create a **Set** node (`Set — Extract Email Fields`). Map assignments:
     - `emailUid` = `={{ $json.uid || $json.id || $json.messageId }}`
     - `emailSubject` = `={{ $json.envelope.subject }}`
     - `emailFrom` = `={{ $json.envelope.from[0].name }}`
     - `emailBody` = `={{ $json.htmlContent }}`
     - `emailDate` = `={{ $json.envelope.date }}`
   - **Step 4:** Create an **Advanced AI Agent** node (`OpenAI Agent — Score Relevance`). Set prompt type to `Define` and paste the legal AI evaluation prompt instructions.
   - **Step 5:** Create an **OpenAI Chat Model** node (`OpenAI — Relevance Model`). Select model `gpt-4o-mini` and provide OpenAI API credentials. Connect its `ai_languageModel` output to the Agent node.
   - **Step 6:** Create a **Code** node (`Code — Parse AI Analysis`). Paste JavaScript code designed to safely parse JSON or fall back to regex extractions.
   - **Step 7:** Create a **Filter** node (`Filter — Keep Relevant Only`). Set condition rule where `={{ $json.isRelevant }}` equals `true`.
   - **Step 8:** Create an **IMAP** node (`IMAP — Delete Processed Email`). Set resource to `email`, operation to `Delete`, and configure UID to `={{ $('IMAP — Fetch Unread Emails').item.json.uid }}` using IMAP credentials.
   - **Step 9:** Create an **Aggregate** node (`Aggregate — Collect Topics`). Set aggregation mode to aggregate all item data.
   - **Step 10:** Create an **If** node (`IF — Has Topics`). Set condition to check if `={{ $json.data.length }}` is greater than `0`.
   - **Step 11:** Create an **Advanced AI Agent** node (`OpenAI Agent — Research Topics`). Set prompt type to `Define` with markdown research reporting constraints.
   - **Step 12:** Create an **OpenAI Chat Model** node (`OpenAI — Research Model`). Select model `gpt-4o` with OpenAI credentials, connecting it via `ai_languageModel`.
   - **Step 13:** Create a **Tavily Web Search** node (`Tavily — Web Search`). Provide Tavily API credentials and set the search query input to `={{ $json.data[0].analysis.primaryTopic }}`, connecting it via `ai_tool`.
   - **Step 14:** Create a **Code** node (`Code — Build HTML Email`). Add script logic parsing markdown tags into inline-styled HTML blocks enveloped inside a responsive email template.
   - **Step 15:** Create a **Gmail** node (`Gmail — Send Digest`). Set operation to `Send`, input target recipient address, configure message body to `={{ $json.htmlEmail }}`, subject to `={{ $json.subject }}`, and append attribution to `false`. Configure Gmail OAuth2 credentials.

3. **Connection Sequence:**
   - `Schedule — Every 5 Hours` ➔ `IMAP — Fetch Unread Emails`
   - `IMAP — Fetch Unread Emails` ➔ `Set — Extract Email Fields`
   - `Set — Extract Email Fields` ➔ `OpenAI Agent — Score Relevance`
   - `OpenAI — Relevance Model` ➔ (AI Language Model link) ➔ `OpenAI Agent — Score Relevance`
   - `OpenAI Agent — Score Relevance` ➔ `Code — Parse AI Analysis`
   - `Code — Parse AI Analysis` (Output 1) ➔ `Filter — Keep Relevant Only`
   - `Code — Parse AI Analysis` (Output 2) ➔ `IMAP — Delete Processed Email`
   - `Filter — Keep Relevant Only` ➔ `Aggregate — Collect Topics`
   - `Aggregate — Collect Topics` ➔ `IF — Has Topics`
   - `IF — Has Topics` (True branch) ➔ `OpenAI Agent — Research Topics`
   - `OpenAI — Research Model` ➔ (AI Language Model link) ➔ `OpenAI Agent — Research Topics`
   - `Tavily — Web Search` ➔ (AI Tool link) ➔ `OpenAI Agent — Research Topics`
   - `OpenAI Agent — Research Topics` ➔ `Code — Build HTML Email`
   - `Code — Build HTML Email` ➔ `Gmail — Send Digest`

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Support contact for workflow inquiries | support@legalgpts.com |
| Project creator website resource | https://automatedintelligentsolutions.com |
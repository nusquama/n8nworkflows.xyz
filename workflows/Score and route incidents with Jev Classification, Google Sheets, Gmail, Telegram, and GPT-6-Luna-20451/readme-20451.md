Score and route incidents with Jev Classification, Google Sheets, Gmail, Telegram, and GPT-6-Luna

https://n8nworkflows.xyz/workflows/score-and-route-incidents-with-jev-classification--google-sheets--gmail--telegram--and-gpt-6-luna-20451


# Score and route incidents with Jev Classification, Google Sheets, Gmail, Telegram, and GPT-6-Luna

### 1. Workflow Overview

This workflow automates production incident management by receiving alerts via webhook, validating and scoring them using Jev Classification, logging decisions to Google Sheets, routing them to the appropriate response team (via Telegram, Gmail, or ticketing queues), and generating monthly trend reports using an AI model.

The logic is grouped into the following functional blocks:
- **1.1 Intake Reception & Validation:** Ingests incident data, normalizes payload fields, and validates that critical metadata (service name and impact text) is present. Unusable reports are flagged and emailed back without scoring.
- **1.2 Severity Scoring & Routing:** Evaluates usable reports against the Jev Classification model, applying confidence thresholds and numeric scores to segment incidents into five operational lanes (SEV1 through SEV4, plus a human-confirmation lane).
- **1.3 Incident Notification & Logging Lanes:** Appends the record to Google Sheets in every lane before dispatching pages (Telegram), stakeholder updates (AI-generated via GPT-6-Luna and Gmail), tickets, or backlog entries.
- **1.4 Monthly Trend Reporting:** Executes via a scheduled cron trigger on the first of each month, reads the incident severity log, aggregates incidents by severity level, and sends an AI-summarized trend brief to the engineering lead.

---

### 2. Block-by-Block Analysis

#### 2.1 Intake Reception & Validation
- **Overview:** Receives raw incident payloads, normalizes essential properties (stamping missing timestamps), and enforces data quality checks to reject incomplete reports before incurring API scoring costs.
- **Nodes Involved:** 
  - `Incident Intake`
  - `Normalize Incident`
  - `Incident Report Usable?`
  - `Mark Unusable Incident Report`
  - `Email Incident Intake Problem`
- **Node Details:**
  - **Incident Intake**
    - *Type and Technical Role:* Webhook node (`n8n-nodes-base.webhook`), acting as the primary entry point for external alerting systems via HTTP POST.
    - *Configuration:* Listens on path `jev-c2-incident-intake` using HTTP method `POST`.
    - *Input/Output:* No inputs; outputs incoming HTTP request body.
    - *Edge Cases:* Unauthenticated endpoint if security headers/tokens are not configured at the gateway level; timeout if downstream services delay responses.
  - **Normalize Incident**
    - *Type and Technical Role:* Set node (`n8n-nodes-base.set`), maps and standardizes incoming webhook fields.
    - *Configuration:* Assigns variables: `incident_id`, `service`, `impact_text`, `affected_users`, `customer_facing`, `detected_at`, and sets `received_at` to `={{ $json.body.received_at || $now.toISO() }}`.
    - *Input/Output:* Input from `Incident Intake`; outputs structured object with normalized fields.
    - *Edge Cases:* Missing optional fields in body can result in undefined values unless defaults are handled.
  - **Incident Report Usable?**
    - *Type and Technical Role:* If node (`n8n-nodes-base.if`), evaluates whether required fields are present.
    - *Configuration:* Condition checks that both `impact_text` and `service` are non-empty.
    - *Input/Output:* Input from `Normalize Incident`; outputs true branch (valid reports) or false branch (unusable reports).
  - **Mark Unusable Incident Report**
    - *Type and Technical Role:* Set node (`n8n-nodes-base.set`), flags missing parameters.
    - *Configuration:* Sets `status` to `not_scored_report_unusable`, lists missing details, and stamps `flagged_at`.
    - *Input/Output:* Input from false branch of `Incident Report Usable?`; outputs error metadata.
  - **Email Incident Intake Problem**
    - *Type and Technical Role:* Gmail node (`n8n-nodes-base.gmail`), notifies administrators of malformed intake data.
    - *Configuration:* Sends text email using Gmail OAuth2 credentials to `user@example.com`.
    - *Input/Output:* Input from `Mark Unusable Incident Report`; outputs email dispatch confirmation.
    - *Failure Types:* SMTP failures, rate limiting, or invalid OAuth2 token refresh.

---

#### 2.2 Severity Scoring & Routing
- **Overview:** Sends valid incident reports to the Jev Classification service to assess blast radius and user impact, subsequently routing the output based on confidence scores and severity level definitions.
- **Nodes Involved:**
  - `Jev Score Incident Severity`
  - `Route By Confidence And Severity`
- **Node Details:**
  - **Jev Score Incident Severity**
    - *Type and Technical Role:* Jev Classification node (`n8n-nodes-jev-classification.jevClassification`), utilizes an AI classification wrapper to score incidents.
    - *Configuration:* Uses model `jev-latest`, operation `score`, confidence threshold `0.65`, 60-second timeout, and max 3 retries. Evaluates severity across four qualitative levels (SEV4 to SEV1).
    - *Key Expressions:* Concatenates service, affected users, customer-facing flag, detection time, and impact text into the analysis text parameter.
    - *Input/Output:* Input from true branch of `Incident Report Usable?`; outputs enrichment object (`jev`) containing level, score, confidence, and `needsReview`.
    - *Failure Types:* API timeouts, OpenRouter/Jev provider outages, or invalid API credentials.
  - **Route By Confidence And Severity**
    - *Type and Technical Role:* Switch node (`n8n-nodes-base.switch`), splits execution into distinct handling lanes.
    - *Configuration:* Evaluates rules sequentially:
      1. Human confirmation if `={{ $json.jev.needsReview }}` is true.
      2. `SEV1 declare incident` if score $\ge 2.5$.
      3. `SEV2 page on-call` if score $\ge 1.5$.
      4. `SEV3 ticket` if score $\ge 0.5$.
      5. `SEV4 backlog` (fallback rule matching true).
    - *Input/Output:* Input from `Jev Score Incident Severity`; outputs to 5 distinct routing paths.

---

#### 2.3 Incident Notification & Logging Lanes
- **Overview:** Standardizes logging across all response tiers by appending incident records to a Google Sheets audit tab *before* dispatching notifications via Telegram, Gmail, or creating support items.
- **Nodes Involved:**
  - *SEV1 Lane:* `SEV1 Declare And Page`, `Log SEV1 Incident`, `Page Incident Commander And On Call`, `Draft Stakeholder Update`, `Stakeholder Update Model`, `Email Stakeholder Update`
  - *SEV2 Lane:* `SEV2 Page Oncall`, `Log SEV2 Incident`, `Page On Call Engineer`
  - *SEV3 Lane:* `SEV3 Create Ticket`, `Log SEV3 Incident`, `Create Ticket Notice`
  - *SEV4 Lane:* `SEV4 Add To Backlog`, `Log SEV4 Incident`
  - *Human Review Lane:* `Severity Needs Human Decision`, `Log Unconfirmed Severity`, `Ask Incident Commander To Confirm`
- **Node Details:**
  - **Logging Nodes (`Log SEV1/2/3/4 Incident`, `Log Unconfirmed Severity`)**
    - *Type and Technical Role:* Google Sheets nodes (`n8n-nodes-base.googleSheets`), append structured data rows.
    - *Configuration:* Operation set to `append`, targeting the "Incident Severity" tab. Maps properties (`incident_id`, `service`, `severity_level`, `severity_score`, `confidence`, `needs_review`, `paging_owner`, `sla_hours`, `action_text`, `status`, `routed_at`).
    - *Input/Output:* Input from respective Switch branches; outputs row insertion confirmation.
    - *Failure Types:* Google Sheets API rate limits, revoked OAuth2 permissions, or missing column headers.
  - **Messaging & Notification Nodes (`Page Incident Commander And On Call`, `Page On Call Engineer`, `Ask Incident Commander To Confirm`, `Create Ticket Notice`, `Email Stakeholder Update`)**
    - *Type and Technical Role:* Telegram (`n8n-nodes-base.telegram`) and Gmail (`n8n-nodes-base.gmail`) nodes for real-time paging and email updates.
    - *Configuration:* Telegram nodes send text messages to a designated `chatId`. Gmail nodes send formatted email updates via OAuth2.
    - *Input/Output:* Inputs from Set or AI Agent nodes; outputs message delivery receipts.
  - **Draft Stakeholder Update**
    - *Type and Technical Role:* Advanced AI Agent node (`@n8n/n8n-nodes-langchain.agent`), utilizes an LLM to compose incident briefs.
    - *Configuration:* System prompt configured for an incident communications lead. Connected to `Stakeholder Update Model` (`gpt-6-luna`).
    - *Input/Output:* Input from `Page Incident Commander And On Call`; outputs generated text (`$json.output`) restricted to 90 words across four lines.

---

#### 2.4 Monthly Trend Reporting
- **Overview:** Automates periodic reliability reporting by aggregating historical logs on the first of each month, generating analytical trend briefs via AI, and emailing them to engineering leadership.
- **Nodes Involved:**
  - `Monthly Incident Trend Trigger`
  - `Read Incident Severity Log`
  - `Summarize Incidents By Level`
  - `Draft Incident Trend Report`
  - `Incident Trend Model`
  - `Email Incident Trend Report`
- **Node Details:**
  - **Monthly Incident Trend Trigger**
    - *Type and Technical Role:* Schedule Trigger node (`n8n-nodes-base.scheduleTrigger`), runs cron-based schedules.
    - *Configuration:* Cron expression `0 8 1 * *` (8:00 AM on the first day of every month).
    - *Input/Output:* Triggers execution flow on schedule.
  - **Read Incident Severity Log**
    - *Type and Technical Role:* Google Sheets node (`n8n-nodes-base.googleSheets`), reads data from the tracking sheet.
    - *Configuration:* Reads all rows from the "Incident Severity" tab.
  - **Summarize Incidents By Level**
    - *Type and Technical Role:* Summarize node (`n8n-nodes-base.summarize`), aggregates data records.
    - *Configuration:* Splits by `severity_level` and summarizes counts of `incident_id`.
  - **Draft Incident Trend Report**
    - *Type and Technical Role:* Advanced AI Agent node (`@n8n/n8n-nodes-langchain.agent`), drafts monthly executive summaries.
    - *Configuration:* System message acts as a reliability engineer. Bound to `Incident Trend Model` (`gpt-6-luna`).
  - **Email Incident Trend Report**
    - *Type and Technical Role:* Gmail node (`n8n-nodes-base.gmail`), distributes the final report.
    - *Configuration:* Mails the AI-generated trend brief text to the engineering lead.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Incident Intake | n8n-nodes-base.webhook | Webhook trigger for incoming incident alerts | None | Normalize Incident | Production incident severity scoring |
| Normalize Incident | n8n-nodes-base.set | Standardizes payload keys and stamps timestamps | Incident Intake | Incident Report Usable? | Production incident severity scoring |
| Incident Report Usable? | n8n-nodes-base.if | Validates presence of service and impact text | Normalize Incident | Jev Score Incident Severity, Mark Unusable Incident Report | Production incident severity scoring |
| Mark Unusable Incident Report | n8n-nodes-base.set | Flags payload errors for rejected reports | Incident Report Usable? | Email Incident Intake Problem | Intake guard |
| Email Incident Intake Problem | n8n-nodes-base.gmail | Emails team about invalid incident reports | Mark Unusable Incident Report | None | Intake guard |
| Jev Score Incident Severity | n8n-nodes-jev-classification.jevClassification | Evaluates severity score and confidence level | Incident Report Usable? | Route By Confidence And Severity | Production incident severity scoring |
| Route By Confidence And Severity | n8n-nodes-base.switch | Routes incident based on review flag and score | Jev Score Incident Severity | Severity Needs Human Decision, SEV1 Declare And Page, SEV2 Page Oncall, SEV3 Create Ticket, SEV4 Add To Backlog | Severity routing and logging |
| Severity Needs Human Decision | n8n-nodes-base.set | Prepares parameters for unconfirmed severity | Route By Confidence And Severity | Log Unconfirmed Severity | Severity routing and logging |
| Log Unconfirmed Severity | n8n-nodes-base.googleSheets | Appends unconfirmed review row to sheet | Severity Needs Human Decision | Ask Incident Commander To Confirm | Severity routing and logging |
| Ask Incident Commander To Confirm | n8n-nodes-base.telegram | Pings commander on Telegram for manual review | Log Unconfirmed Severity | None | Severity routing and logging |
| SEV1 Declare And Page | n8n-nodes-base.set | Prepares parameters for SEV1 critical incidents | Route By Confidence And Severity | Log SEV1 Incident | Severity routing and logging |
| Log SEV1 Incident | n8n-nodes-base.googleSheets | Logs SEV1 incident to tracking sheet | SEV1 Declare And Page | Page Incident Commander And On Call | Severity routing and logging |
| Page Incident Commander And On Call | n8n-nodes-base.telegram | Sends high-urgency Telegram alert for SEV1 | Log SEV1 Incident | Draft Stakeholder Update | Severity routing and logging |
| Draft Stakeholder Update | @n8n/n8n-nodes-langchain.agent | Generates concise stakeholder update via AI | Page Incident Commander And On Call | Email Stakeholder Update | Severity routing and logging |
| Stakeholder Update Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides LLM backend (`gpt-6-luna`) for stakeholder updates | None | Draft Stakeholder Update | Severity routing and logging |
| Email Stakeholder Update | n8n-nodes-base.gmail | Sends stakeholder update email | Draft Stakeholder Update | None | Severity routing and logging |
| SEV2 Page Oncall | n8n-nodes-base.set | Prepares parameters for SEV2 major incidents | Route By Confidence And Severity | Log SEV2 Incident | Severity routing and logging |
| Log SEV2 Incident | n8n-nodes-base.googleSheets | Logs SEV2 incident to tracking sheet | SEV2 Page Oncall | Page On Call Engineer | Severity routing and logging |
| Page On Call Engineer | n8n-nodes-base.telegram | Pages on-call engineer via Telegram | Log SEV2 Incident | None | Severity routing and logging |
| SEV3 Create Ticket | n8n-nodes-base.set | Prepares parameters for SEV3 minor incidents | Route By Confidence And Severity | Log SEV3 Incident | Severity routing and logging |
| Log SEV3 Incident | n8n-nodes-base.googleSheets | Logs SEV3 incident to tracking sheet | SEV3 Create Ticket | Create Ticket Notice | Severity routing and logging |
| Create Ticket Notice | n8n-nodes-base.gmail | Sends service desk ticket notification email | Log SEV3 Incident | None | Severity routing and logging |
| SEV4 Add To Backlog | n8n-nodes-base.set | Prepares parameters for SEV4 negligible incidents | Route By Confidence And Severity | Log SEV4 Incident | Severity routing and logging |
| Log SEV4 Incident | n8n-nodes-base.googleSheets | Appends backlog item to sheet | SEV4 Add To Backlog | None | Severity routing and logging |
| Monthly Incident Trend Trigger | n8n-nodes-base.scheduleTrigger | Monthly cron trigger for trend reporting | None | Read Incident Severity Log | Monthly trend lane |
| Read Incident Severity Log | n8n-nodes-base.googleSheets | Reads incident history for aggregation | Monthly Incident Trend Trigger | Summarize Incidents By Level | Monthly trend lane |
| Summarize Incidents By Level | n8n-nodes-base.summarize | Aggregates incident counts by severity band | Read Incident Severity Log | Draft Incident Trend Report | Monthly trend lane |
| Draft Incident Trend Report | @n8n/n8n-nodes-langchain.agent | Generates monthly trend brief via AI | Summarize Incidents By Level | Email Incident Trend Report | Monthly trend lane |
| Incident Trend Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides LLM backend (`gpt-6-luna`) for monthly trend reports | None | Draft Incident Trend Report | Monthly trend lane |
| Email Incident Trend Report | n8n-nodes-base.gmail | Emails trend report to engineering lead | Draft Incident Trend Report | None | Monthly trend lane |

---

### 4. Reproducing the Workflow from Scratch

Follow these manual steps to rebuild the workflow:

1. **Setup Google Sheets:** Create a new Google Spreadsheet containing a tab named exactly `Incident Severity`. Add the following column headers in row 1: `incident_id`, `received_at`, `service`, `severity_level`, `severity_score`, `confidence`, `needs_review`, `paging_owner`, `sla_hours`, `action_taken`, `impact_excerpt`, `status`, and `routed_at`.
2. **Create Intake Trigger:** Add a **Webhook** node named `Incident Intake`. Set HTTP Method to `POST` and path to `jev-c2-incident-intake`.
3. **Normalize Data:** Connect a **Set** node named `Normalize Incident`. Map payload parameters (`incident_id`, `service`, `impact_text`, `affected_users`, `customer_facing`, `detected_at`) and assign `received_at` with expression `={{ $json.body.received_at || $now.toISO() }}`.
4. **Validation Check:** Connect an **If** node named `Incident Report Usable?`. Create conditions verifying that both `impact_text` and `service` are not empty.
5. **Handle Unusable Reports:** 
   - From the *false* branch, connect a **Set** node named `Mark Unusable Incident Report` to capture missing properties and set status to `not_scored_report_unusable`.
   - Connect a **Gmail** node named `Email Incident Intake Problem` configured with Gmail OAuth2 credentials to send error alerts back to the intake address.
6. **Configure Classification:** 
   - From the *true* branch of the validator, connect a **Jev Classification** node named `Score Incident Severity`. Configure model `jev-latest`, set operation to `score`, set confidence threshold to `0.65`, and configure OpenRouter/JEV API credentials.
7. **Add Routing Switch:** Connect a **Switch** node named `Route By Confidence And Severity` with 5 rules matching:
   - Output 0: `={{ $json.jev.needsReview }}` equals `true` (Human confirms severity).
   - Output 1: `={{ $json.jev.score }}` $\ge 2.5$ (SEV1).
   - Output 2: `={{ $json.jev.score }}` $\ge 1.5$ (SEV2).
   - Output 3: `={{ $json.jev.score }}` $\ge 0.5$ (SEV3).
   - Output 4: Fallback true (SEV4).
8. **Build Escalation Lanes (SEV1 to SEV4 & Human Confirm):**
   - For each route, add a **Set** node (`SEV1 Declare And Page`, `SEV2 Page Oncall`, `SEV3 Create Ticket`, `SEV4 Add To Backlog`, `Severity Needs Human Decision`) defining operational SLAs, priorities, and audit statuses.
   - Attach a corresponding **Google Sheets** node (`Log SEV1/2/3/4 Incident`, `Log Unconfirmed Severity`) set to append rows to your spreadsheet tab.
   - Attach notification nodes: **Telegram** nodes (`Page Incident Commander And On Call`, `Page On Call Engineer`, `Ask Incident Commander To Confirm`) and **Gmail** notification nodes (`Create Ticket Notice`, `Email Stakeholder Update`).
   - For the SEV1 stakeholder update, configure an **Advanced AI Agent** node (`Draft Stakeholder Update`) powered by an OpenAI Language Model node (`Stakeholder Update Model`) using model `gpt-6-luna`.
9. **Build Monthly Reporting Cron:**
   - Create a **Schedule Trigger** node (`Monthly Incident Trend Trigger`) set to expression `0 8 1 * *`.
   - Connect a **Google Sheets** node (`Read Incident Severity Log`) to read the "Incident Severity" tab.
   - Connect a **Summarize** node (`Summarize Incidents By Level`) splitting by `severity_level` and counting `incident_id`.
   - Connect an **Advanced AI Agent** node (`Draft Incident Trend Report`) linked to an OpenAI Language Model node (`Incident Trend Model`) using model `gpt-6-luna`.
   - Conclude with a **Gmail** node (`Email Incident Trend Report`) to deliver the finalized brief to the engineering lead.
10. **Finalize Credentials & Testing:** Configure active OAuth2 credentials for Google Sheets and Gmail, API keys for Telegram bots and OpenAI/OpenRouter, verify timezone settings, and activate the workflow.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Built with n8n workflow automation platform | [n8n Creator Partner Link](https://n8n.partnerlinks.io/creator-khmuhtadin) |
| Professional consultation and business assessments | [Consultation Request Page](https://khmuhtadin.com/consultation/) |
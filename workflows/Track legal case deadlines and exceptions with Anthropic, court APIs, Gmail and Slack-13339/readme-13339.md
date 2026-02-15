Track legal case deadlines and exceptions with Anthropic, court APIs, Gmail and Slack

https://n8nworkflows.xyz/workflows/track-legal-case-deadlines-and-exceptions-with-anthropic--court-apis--gmail-and-slack-13339


# Track legal case deadlines and exceptions with Anthropic, court APIs, Gmail and Slack

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title (given):** Track legal case deadlines and exceptions with Anthropic, court APIs, Gmail and Slack  
**Workflow name (JSON):** AI-powered legal case management and deadline monitoring automation

**Purpose:**  
Poll a court/case-management API every 15 minutes, validate administrative integrity of case data with an Anthropic-powered agent, detect exceptions (incomplete/conflicting/error states), and orchestrate administrative responses: deadline tracking, exception escalation, and notifications via Gmail and Slack. Finally, store validated cases and exception cases separately.

**Target use cases:**
- Litigation deadline tracking and docket monitoring
- Court filing completeness checks and sequencing validation
- Exception escalation for conflicts, missing fields, or critical deadline proximity
- Administrative coordination (not legal merits)

### Logical blocks
**1.1 Scheduled Intake & Configuration**  
Schedule trigger → set configuration variables → fetch case data from court API.

**1.2 AI Validation (Administrative Integrity Only)**  
Send raw case payload to a validation agent (Anthropic) with a structured output parser.

**1.3 Exception Gate & Orchestrated Response**  
If validation indicates escalation is required, run an orchestrator agent that can call two AI tools (deadline tracking, exception escalation) and two notification tools (Gmail, Slack).

**1.4 Routing & Storage**  
Route by `validationStatus` and store either “validated” or “exception” records into n8n Data Tables.

---

## 2. Block-by-Block Analysis

### 2.1 Scheduled Intake & Configuration

**Overview:**  
Triggers periodically, loads workflow constants (API URL, thresholds, recipients), then retrieves court case data via HTTP.

**Nodes involved:**
- Schedule Trigger - Every 15 Minutes
- Workflow Configuration
- Fetch Court Case Data

#### Node: Schedule Trigger - Every 15 Minutes
- **Type / role:** `n8n-nodes-base.scheduleTrigger` — entry point; runs on interval.
- **Configuration choices:** Interval rule: every **15 minutes**.
- **Inputs/outputs:** No input. Main output → **Workflow Configuration**.
- **Edge cases / failures:** Missed executions if n8n is down; drift depending on instance load; time-sensitive deadlines might need shorter intervals.
- **Version notes:** TypeVersion **1.3**.

#### Node: Workflow Configuration
- **Type / role:** `n8n-nodes-base.set` — defines constants used downstream.
- **Key fields created:**
  - `courtApiUrl` (placeholder)
  - `validationThresholdDays` = 7
  - `criticalDeadlineDays` = 3
  - `notificationEmail` (placeholder)
  - `slackChannel` (placeholder)
- **Config choices:** “Include other fields” enabled (keeps incoming trigger fields).
- **Expressions/variables:** Values are plain except placeholders. Downstream reference uses `$('Workflow Configuration').item.json...`.
- **Connections:** Input from Trigger. Output → **Fetch Court Case Data**.
- **Edge cases:** Placeholder values not replaced → HTTP node fails; thresholds unused unless agents incorporate them (currently prompts encode rules; variables aren’t injected into prompts).
- **Version notes:** TypeVersion **3.4**.

#### Node: Fetch Court Case Data
- **Type / role:** `n8n-nodes-base.httpRequest` — retrieves court/case payload.
- **Configuration choices:**
  - URL: `={{ $('Workflow Configuration').item.json.courtApiUrl }}`
  - Sends header `Content-Type: application/json`
  - `sendHeaders: true`
- **Connections:** Input from Workflow Configuration. Output → **Caseflow Validation Agent**.
- **Edge cases / failures:**
  - Auth missing (no auth configured here) if API requires it (likely).
  - Non-200 responses; schema drift; pagination not handled (if API returns multiple pages).
  - Timeout / rate limiting / IP restrictions common with court systems.
- **Version notes:** TypeVersion **4.3**.

---

### 2.2 AI Validation (Administrative Integrity Only)

**Overview:**  
Validates completeness, scheduling integrity, procedural transition consistency, and flags conflicts—explicitly excluding legal merits.

**Nodes involved:**
- Caseflow Validation Agent
- Anthropic Model - Validation Agent
- Validation Output Parser

#### Node: Caseflow Validation Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — LLM agent performing validation.
- **Configuration choices:**
  - Input text: `={{ $json }}` (passes the HTTP response JSON as the agent message).
  - System message strictly limits scope to administrative validation, returns structured fields:
    - `validationStatus` (VALID/INCOMPLETE/CONFLICT/ERROR)
    - `caseId`, `issues`, `missingFields`, `deadlineConflicts`, `nextDeadline`, `requiresEscalation`
  - **Has output parser enabled**.
- **Model connection:** Uses **Anthropic Model - Validation Agent** as `ai_languageModel`.
- **Output parser connection:** Uses **Validation Output Parser**.
- **Main connections:** Output → **Check for Exceptions**.
- **Edge cases / failures:**
  - LLM may return non-conforming JSON → parser failure (execution error).
  - If the court API returns an array of cases, passing whole array as `$json` may cause token bloat; consider itemizing.
  - `caseId` might not exist in source; agent inventing IDs is possible unless grounded by strict instructions.
- **Version notes:** TypeVersion **3.1** (LangChain agent node).

#### Node: Anthropic Model - Validation Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatAnthropic` — language model provider for validation agent.
- **Configuration choices:** Model `claude-sonnet-4-5-20250929` (cached name “Claude Sonnet 4.5”).
- **Credentials:** Anthropic API credential (named “Anthropic account”).
- **Connections:** Provides `ai_languageModel` to **Caseflow Validation Agent**.
- **Edge cases:** Invalid API key, billing limits, model deprecation/renaming, latency/timeouts.
- **Version notes:** TypeVersion **1.3**.

#### Node: Validation Output Parser
- **Type / role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — enforces a structured JSON shape.
- **Configuration choices:** JSON schema example (used as guidance) includes all expected fields.
- **Connections:** Attached to **Caseflow Validation Agent** as `ai_outputParser`.
- **Edge cases:** If agent outputs types incorrectly (e.g., `requiresEscalation` as `"false"` string), parser may fail or coerce depending on node behavior/version.
- **Version notes:** TypeVersion **1.3**.

---

### 2.3 Exception Gate & Orchestrated Response

**Overview:**  
Checks whether escalation is required, then uses an orchestration agent to call specialized tools: deadline tracking and exception escalation, plus Gmail and Slack notification tools.

**Nodes involved:**
- Check for Exceptions
- Administration Orchestrator Agent
- Anthropic Model - Admin Agent
- Deadline Tracking Agent Tool
- Anthropic Model - Deadline Agent
- Deadline Tracking Output Parser
- Exception Escalation Agent Tool
- Anthropic Model - Exception Agent
- Exception Escalation Output Parser
- Gmail Notification Tool
- Slack Alert Tool

#### Node: Check for Exceptions
- **Type / role:** `n8n-nodes-base.if` — gates exception escalation path.
- **Condition:** `={{ $json.output.requiresEscalation }}` **equals** `true`
  - Note: it checks `$json.output.requiresEscalation` (nested under `output`).
- **Connections (important):**
  - **True path (Output 0)** → **Administration Orchestrator Agent**
  - **False path (Output 1)** → **Route by Validation Status**
- **Edge cases / failures:**
  - If the parsed validation result is not under `output`, condition will evaluate false or error. Many n8n AI nodes output structured results either at top-level or under `output` depending on node/version and settings. This is a common break point.
  - If `requiresEscalation` is missing/null → condition fails and case routes as non-exception even when it should escalate.
- **Version notes:** TypeVersion **2.3**.

#### Node: Administration Orchestrator Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — multi-tool agent that decides which tools to call and crafts notifications.
- **Input text:** `={{ $json }}` (receives item from IF true branch).
- **System message highlights:**
  - Must call **Deadline Tracking Agent Tool** first.
  - Then call **Exception Escalation Agent Tool** if exceptions exist.
  - Notification rules:
    - Critical deadlines (<3 days): email + Slack
    - Standard deadlines (3–7 days): email only
    - Exceptions needing escalation: Slack immediately
- **Tooling connections (AI tools):**
  - **Deadline Tracking Agent Tool** (ai_tool)
  - **Exception Escalation Agent Tool** (ai_tool)
  - **Gmail Notification Tool** (ai_tool)
  - **Slack Alert Tool** (ai_tool)
- **Model connection:** **Anthropic Model - Admin Agent**.
- **Main output:** → **Route by Validation Status**
- **Edge cases / failures:**
  - Agent may not follow ordering/tool-calling instructions.
  - If it calls Gmail/Slack with missing parameters (recipient/channel/message), tool nodes will error.
  - Threshold values (3,7 days) are hard-coded in the prompt; the Set node thresholds aren’t injected, so changing them in config won’t affect orchestration unless prompt is updated.
- **Version notes:** TypeVersion **3.1**.

#### Node: Anthropic Model - Admin Agent
- **Type / role:** Anthropic chat model for orchestrator.
- **Model:** `claude-sonnet-4-5-20250929`
- **Credentials:** Anthropic API.
- **Connections:** `ai_languageModel` → Administration Orchestrator Agent.
- **Edge cases:** Same as other Anthropic nodes (quota, timeouts, model availability).
- **Version notes:** TypeVersion **1.3**.

#### Node: Deadline Tracking Agent Tool
- **Type / role:** `@n8n/n8n-nodes-langchain.agentTool` — tool callable by the orchestrator; performs deadline computations and urgency classification.
- **Input mapping:**  
  `={{ $fromAI('caseData', 'Validated case data for deadline analysis', 'json') }}`
  - This means the **orchestrator** must provide a `caseData` JSON argument when calling this tool.
- **System message outputs:**
  - `caseId`, `deadlines[]` with `{date,type,daysUntil,urgency}`, `conflicts[]`, `criticalCount`, `recommendations[]`
- **Model connection:** **Anthropic Model - Deadline Agent**
- **Output parser:** **Deadline Tracking Output Parser**
- **Connections:** Exposed as `ai_tool` to Administration Orchestrator Agent (not in main flow).
- **Edge cases / failures:**
  - If orchestrator passes malformed `caseData`, tool can hallucinate or fail parsing.
  - Date parsing/timezones: “daysUntil” depends on server time and date format; prompt doesn’t specify timezone normalization.
- **Version notes:** TypeVersion **3**.

#### Node: Anthropic Model - Deadline Agent
- **Type / role:** Anthropic chat model for the deadline tool.
- **Model:** `claude-sonnet-4-5-20250929`
- **Credentials:** Anthropic API.
- **Connections:** `ai_languageModel` → Deadline Tracking Agent Tool.
- **Version notes:** TypeVersion **1.3**.

#### Node: Deadline Tracking Output Parser
- **Type / role:** Structured output parser for deadline tool.
- **Schema example:** includes `deadlines[]` objects and counts.
- **Connections:** `ai_outputParser` → Deadline Tracking Agent Tool.
- **Edge cases:** Tool output not matching schema → parser failure.
- **Version notes:** TypeVersion **1.3**.

#### Node: Exception Escalation Agent Tool
- **Type / role:** `@n8n/n8n-nodes-langchain.agentTool` — tool callable by orchestrator; classifies exceptions and escalation paths.
- **Input mapping:**  
  `={{ $fromAI('exceptionData', 'Exception data requiring escalation analysis', 'json') }}`
  - Orchestrator must provide `exceptionData` JSON when calling.
- **Outputs:** `exceptionId`, `severity`, `escalationPath`, `stakeholders[]`, `responseTimeframe`, `recommendations[]`, `summary`
- **Model connection:** **Anthropic Model - Exception Agent**
- **Output parser:** **Exception Escalation Output Parser**
- **Connections:** Exposed as `ai_tool` to Administration Orchestrator Agent.
- **Edge cases:** Same pattern: incorrect/missing `exceptionData`; parser mismatch.
- **Version notes:** TypeVersion **3**.

#### Node: Anthropic Model - Exception Agent
- **Type / role:** Anthropic chat model for exception tool.
- **Model:** `claude-sonnet-4-5-20250929`
- **Credentials:** Anthropic API.
- **Connections:** `ai_languageModel` → Exception Escalation Agent Tool.
- **Version notes:** TypeVersion **1.3**.

#### Node: Exception Escalation Output Parser
- **Type / role:** Structured output parser for exception escalation tool.
- **Connections:** `ai_outputParser` → Exception Escalation Agent Tool.
- **Edge cases:** If stakeholders are not valid emails/user IDs, downstream notifications can fail.
- **Version notes:** TypeVersion **1.3**.

#### Node: Gmail Notification Tool
- **Type / role:** `n8n-nodes-base.gmailTool` — callable tool to send email.
- **Tool parameters (all AI-provided via `$fromAI`):**
  - `sendTo`: `recipientEmail` (string)
  - `subject`: `emailSubject` (string)
  - `message`: `emailBody` (HTML string)
- **Credentials:** Gmail OAuth2 (“Gmail account”).
- **Connections:** Exposed as `ai_tool` to Administration Orchestrator Agent.
- **Edge cases / failures:**
  - OAuth token expiry / missing scopes (send mail).
  - Invalid email address from AI.
  - HTML body may be malformed; deliverability concerns.
- **Version notes:** TypeVersion **2.2**.

#### Node: Slack Alert Tool
- **Type / role:** `n8n-nodes-base.slackTool` — callable tool to post Slack messages.
- **Parameters (AI-provided):**
  - `text`: `alertMessage`
  - `channelId`: `slackChannel`
  - `authentication`: OAuth2
- **Credentials:** Slack OAuth2 (“Slack account”).
- **Connections:** Exposed as `ai_tool` to Administration Orchestrator Agent.
- **Edge cases / failures:**
  - Missing `chat:write` scope, channel not found, bot not in channel.
  - AI supplies a channel name instead of ID → failure.
- **Version notes:** TypeVersion **2.4**.

---

### 2.4 Routing & Storage

**Overview:**  
Routes each case based on validation result and stores it in the appropriate n8n Data Table.

**Nodes involved:**
- Route by Validation Status
- Store Validated Cases
- Store Exceptions

#### Node: Route by Validation Status
- **Type / role:** `n8n-nodes-base.switch` — routes by `validationStatus`.
- **Rules:**
  - Output “Valid Cases” if `{{$json.validationStatus}} == "VALID"`
  - Output “Exception Cases” if status is `"INCOMPLETE"` or `"CONFLICT"` or `"ERROR"`
  - Fallback output renamed to “Unknown Status”
- **Inputs:**
  - From **Check for Exceptions** false path (non-escalations)
  - From **Administration Orchestrator Agent** (post-orchestration)
- **Outputs:**
  - “Valid Cases” → Store Validated Cases
  - “Exception Cases” → Store Exceptions
- **Edge cases:**
  - If the actual status is under `$json.output.validationStatus` (common with AI nodes), routing fails and goes to fallback “Unknown Status” (currently unconnected → data dropped).
  - Duplicate outputKey “Exception Cases” appears as three separate rules; n8n supports multiple rules pointing to the same named output, but maintenance can be confusing.
- **Version notes:** TypeVersion **3.4**.

#### Node: Store Validated Cases
- **Type / role:** `n8n-nodes-base.dataTable` — persists valid cases.
- **Configuration choices:** Auto-map input data to columns.
- **Data table selection:** `dataTableId` is empty (must be selected).
- **Connections:** Input from Route by Validation Status (Valid Cases). No outputs.
- **Edge cases / failures:**
  - No Data Table selected → node errors.
  - Schema drift: auto-mapping may create inconsistent columns over time.
- **Version notes:** TypeVersion **1.1**.

#### Node: Store Exceptions
- **Type / role:** `n8n-nodes-base.dataTable` — persists exception cases.
- **Configuration choices:** Auto-map input data to columns.
- **Data table selection:** `dataTableId` is empty (must be selected).
- **Connections:** Input from Route by Validation Status (Exception Cases). No outputs.
- **Edge cases / failures:** Same as Store Validated Cases.
- **Version notes:** TypeVersion **1.1**.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger - Every 15 Minutes | scheduleTrigger | Periodic workflow entry | — | Workflow Configuration | ## How It Works\nThis workflow automates legal case tracking, deadline management, and exception handling...\nThe system schedules regular monitoring (every 15 minutes...) ... reduce missed deadline risk by 95%... |
| Workflow Configuration | set | Define constants (API URL, thresholds, recipients) | Schedule Trigger - Every 15 Minutes | Fetch Court Case Data | ## How It Works\nThis workflow automates legal case tracking... |
| Fetch Court Case Data | httpRequest | Pull case data from court API | Workflow Configuration | Caseflow Validation Agent | ## AI Case Classification\n**What**: Classifier Agent categorizes cases by type and urgency; Validation Agent confirms data accuracy\n**Why**: Intelligent triage prioritizes time-sensitive matters... |
| Caseflow Validation Agent | langchain.agent | Validate administrative integrity and produce structured status | Fetch Court Case Data | Check for Exceptions | ## AI Case Classification\n**What**: Classifier Agent categorizes cases by type and urgency; Validation Agent confirms data accuracy\n**Why**: Intelligent triage prioritizes time-sensitive matters... |
| Validation Output Parser | langchain.outputParserStructured | Enforce validation JSON shape | Caseflow Validation Agent (parser connection) | — | ## AI Case Classification\n**What**: Classifier Agent categorizes cases by type and urgency; Validation Agent confirms data accuracy\n**Why**: Intelligent triage prioritizes time-sensitive matters... |
| Anthropic Model - Validation Agent | lmChatAnthropic | LLM backend for validation | — | Caseflow Validation Agent | ## AI Case Classification\n**What**: Classifier Agent categorizes cases by type and urgency; Validation Agent confirms data accuracy\n**Why**: Intelligent triage prioritizes time-sensitive matters... |
| Check for Exceptions | if | Gate: requiresEscalation? | Caseflow Validation Agent | Administration Orchestrator Agent; Route by Validation Status | ## Exception Detection\n**What**: Checks for critical exceptions—missed deadlines, conflicting dates, and incomplete filings\n**Why**: Early exception identification enables corrective action... |
| Administration Orchestrator Agent | langchain.agent | Coordinate tools and notifications | Check for Exceptions (true) | Route by Validation Status | ## Orchestrated Response Coordination\n**What**: Administration Agent coordinates deadline tracking, exception escalation via Gmail/Slack, and case documentation\n**Why**: Multi-agent coordination ensures timely stakeholder notification... |
| Anthropic Model - Admin Agent | lmChatAnthropic | LLM backend for orchestrator | — | Administration Orchestrator Agent | ## Orchestrated Response Coordination\n**What**: Administration Agent coordinates deadline tracking... |
| Deadline Tracking Agent Tool | langchain.agentTool | Tool: compute deadline urgency/conflicts | Administration Orchestrator Agent (tool call) | — | ## Orchestrated Response Coordination\n**What**: Administration Agent coordinates deadline tracking... |
| Deadline Tracking Output Parser | langchain.outputParserStructured | Enforce deadline tool JSON shape | Deadline Tracking Agent Tool (parser connection) | — | ## Orchestrated Response Coordination\n**What**: Administration Agent coordinates deadline tracking... |
| Anthropic Model - Deadline Agent | lmChatAnthropic | LLM backend for deadline tool | — | Deadline Tracking Agent Tool | ## Orchestrated Response Coordination\n**What**: Administration Agent coordinates deadline tracking... |
| Exception Escalation Agent Tool | langchain.agentTool | Tool: classify exception severity/escalation | Administration Orchestrator Agent (tool call) | — | ## Orchestrated Response Coordination\n**What**: Administration Agent coordinates deadline tracking... |
| Exception Escalation Output Parser | langchain.outputParserStructured | Enforce exception tool JSON shape | Exception Escalation Agent Tool (parser connection) | — | ## Orchestrated Response Coordination\n**What**: Administration Agent coordinates deadline tracking... |
| Anthropic Model - Exception Agent | lmChatAnthropic | LLM backend for exception tool | — | Exception Escalation Agent Tool | ## Orchestrated Response Coordination\n**What**: Administration Agent coordinates deadline tracking... |
| Gmail Notification Tool | gmailTool | Tool: send email notifications | Administration Orchestrator Agent (tool call) | — | ## Orchestrated Response Coordination\n**What**: Administration Agent coordinates deadline tracking... |
| Slack Alert Tool | slackTool | Tool: post urgent Slack alerts | Administration Orchestrator Agent (tool call) | — | ## Orchestrated Response Coordination\n**What**: Administration Agent coordinates deadline tracking... |
| Route by Validation Status | switch | Route VALID vs exception statuses | Check for Exceptions (false); Administration Orchestrator Agent | Store Validated Cases; Store Exceptions | ## How It Works\nThis workflow automates legal case tracking... |
| Store Validated Cases | dataTable | Persist valid cases | Route by Validation Status (Valid Cases) | — |  |
| Store Exceptions | dataTable | Persist exception cases | Route by Validation Status (Exception Cases) |  |  |
| Sticky Note | stickyNote | Notes: prerequisites/use cases/customization/benefits | — | — | ## Prerequisites\nCourt system API access (PACER, state portals), case management system integration\n## Use Cases\nLitigation deadline tracking...\n## Customization\nModify classification rules...\n## Benefits\nReduces missed deadline risk by 95%... |
| Sticky Note1 | stickyNote | Notes: setup steps | — | — | ## Setup Steps\n1. Connect **Schedule Trigger**...\n7. Configure **Slack** webhooks... |
| Sticky Note2 | stickyNote | Notes: high-level explanation | — | — | ## How It Works\nThis workflow automates legal case tracking... |
| Sticky Note3 | stickyNote | Notes: exception detection | — | — | ## Exception Detection\n**What**: Checks for critical exceptions...\n**Why**: Early exception identification... |
| Sticky Note4 | stickyNote | Notes: AI classification/validation | — | — | ## AI Case Classification\n**What**: Classifier Agent categorizes...\n**Why**: Intelligent triage... |
| Sticky Note5 | stickyNote | Notes: orchestration | — | — | ## Orchestrated Response Coordination\n**What**: Administration Agent coordinates...\n**Why**: Multi-agent coordination... |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Trigger**
1. Add node **Schedule Trigger**.
2. Set interval to **Every 15 minutes**.

2) **Add configuration node**
3. Add node **Set** named “Workflow Configuration”.
4. Add fields:
   - `courtApiUrl` (string) → your court/case API endpoint
   - `validationThresholdDays` (number) = 7
   - `criticalDeadlineDays` (number) = 3
   - `notificationEmail` (string) → admin recipient (optional unless used)
   - `slackChannel` (string) → Slack channel ID (optional unless used)
5. Enable “Include other fields” (optional).

3) **Fetch case data**
6. Add node **HTTP Request** named “Fetch Court Case Data”.
7. Set **URL** to expression: `{{$('Workflow Configuration').item.json.courtApiUrl}}`.
8. Add header `Content-Type: application/json`.
9. Configure authentication as required by your court API (API key/OAuth2/mTLS) in the node’s Auth section.

4) **Validation agent (Anthropic)**
10. Add node **Anthropic Chat Model** named “Anthropic Model - Validation Agent”.
11. Select model `claude-sonnet-4-5-20250929` (or closest available).
12. Create/select **Anthropic API credentials**.

13. Add node **AI Agent (LangChain)** named “Caseflow Validation Agent”.
14. Set input text to `{{$json}}`.
15. Paste the provided **system message** (the validation scope + required output fields).
16. Attach **Anthropic Model - Validation Agent** to the agent’s **Language Model** connection.

17. Add node **Structured Output Parser** named “Validation Output Parser”.
18. Provide a schema example matching:
   - `validationStatus`, `caseId`, `issues[]`, `missingFields[]`, `deadlineConflicts[]`, `nextDeadline`, `requiresEscalation`
19. Connect parser to the agent’s **Output Parser** connection.

5) **Exception gate**
20. Add node **IF** named “Check for Exceptions”.
21. Condition: Boolean equals `true` on expression `{{$json.output.requiresEscalation}}`.
   - If your agent outputs fields at top-level, change to `{{$json.requiresEscalation}}`.

6) **Orchestrator agent + tools**
22. Add node **Anthropic Chat Model** named “Anthropic Model - Admin Agent” and set credentials/model.

23. Add node **AI Agent (LangChain)** named “Administration Orchestrator Agent”.
24. Set input to `{{$json}}`.
25. Paste the orchestrator system message (tool ordering + notification rules).
26. Connect **Anthropic Model - Admin Agent** to its **Language Model** connection.

27. Add node **Anthropic Chat Model** named “Anthropic Model - Deadline Agent”.
28. Add node **Agent Tool** named “Deadline Tracking Agent Tool”.
29. Tool input: `{{$fromAI('caseData','Validated case data for deadline analysis','json')}}`
30. Paste its system message and enable structured output parsing.
31. Add **Structured Output Parser** “Deadline Tracking Output Parser” with the deadline schema example; connect to tool.
32. Connect **Anthropic Model - Deadline Agent** to the tool’s language model input.
33. Connect **Deadline Tracking Agent Tool** to **Administration Orchestrator Agent** via **AI Tool** connection.

34. Add node **Anthropic Chat Model** named “Anthropic Model - Exception Agent”.
35. Add node **Agent Tool** named “Exception Escalation Agent Tool”.
36. Tool input: `{{$fromAI('exceptionData','Exception data requiring escalation analysis','json')}}`
37. Add **Structured Output Parser** “Exception Escalation Output Parser”; connect to tool.
38. Connect **Anthropic Model - Exception Agent** to the tool’s language model input.
39. Connect **Exception Escalation Agent Tool** to **Administration Orchestrator Agent** via **AI Tool** connection.

40. Add node **Gmail Tool** named “Gmail Notification Tool”.
41. Configure Gmail OAuth2 credentials (scopes to send email).
42. Map:
   - To: `{{$fromAI('recipientEmail', ... , 'string')}}`
   - Subject: `{{$fromAI('emailSubject', ... , 'string')}}`
   - Message: `{{$fromAI('emailBody', ... , 'string')}}`
43. Connect Gmail tool to **Administration Orchestrator Agent** via **AI Tool** connection.

44. Add node **Slack Tool** named “Slack Alert Tool”.
45. Configure Slack OAuth2 credentials (scope `chat:write`; bot in channel).
46. Set “Send to” = channel; Channel ID: `{{$fromAI('slackChannel', ... , 'string')}}`
47. Text: `{{$fromAI('alertMessage', ... , 'string')}}`
48. Connect Slack tool to **Administration Orchestrator Agent** via **AI Tool** connection.

7) **Routing and persistence**
49. Add node **Switch** named “Route by Validation Status”.
50. Add rules:
   - If `{{$json.validationStatus}}` equals `VALID` → output “Valid Cases”
   - If equals `INCOMPLETE`/`CONFLICT`/`ERROR` → output “Exception Cases”
   - Add fallback output “Unknown Status” (optional: connect to logging).

51. Add **Data Table** node “Store Validated Cases”; select/create a Data Table for valid cases; set mapping mode to auto-map.
52. Add **Data Table** node “Store Exceptions”; select/create a Data Table for exceptions; set mapping mode to auto-map.

8) **Connect nodes (main flow)**
53. Schedule Trigger → Workflow Configuration → Fetch Court Case Data → Caseflow Validation Agent → Check for Exceptions
54. IF true → Administration Orchestrator Agent → Route by Validation Status
55. IF false → Route by Validation Status
56. Switch “Valid Cases” → Store Validated Cases
57. Switch “Exception Cases” → Store Exceptions

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Court system API access (PACER, state portals), case management system integration | Prerequisites (Sticky Note) |
| Use cases: litigation deadline tracking, court filing monitoring, statute of limitations management | Prerequisites/Use cases (Sticky Note) |
| Customization: modify classification rules for practice areas (patent, corporate, criminal) | Customization (Sticky Note) |
| Benefits claim: “Reduces missed deadline risk by 95%” | Benefits (Sticky Note / Sticky Note2) |
| Setup steps include connecting schedule, configuring court data sources, adding AI keys, defining jurisdiction rules, thresholds, Gmail/Slack setup | Setup guidance (Sticky Note1) |
| Exception detection intent: missed deadlines, conflicting dates, incomplete filings | Concept note (Sticky Note3) |
| AI role separation: validation vs orchestration; do not assess merits | Concept note (Sticky Note4/5) |


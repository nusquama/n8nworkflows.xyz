Analyze public consultation data and coordinate governance with Claude AI

https://n8nworkflows.xyz/workflows/analyze-public-consultation-data-and-coordinate-governance-with-claude-ai-13335


# Analyze public consultation data and coordinate governance with Claude AI

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Smart Public Consultation Analysis and Governance Coordination System  
**Purpose:** Automatically fetch public consultation submissions, analyze each submission with Claude (Anthropic) into structured fields, aggregate results, run governance coordination (bias checks, summary generation, traceability), route outcomes by escalation level, and prepare consolidated traceability records for auditability.

**Target use cases**
- Public-sector consultation processing (sentiment/theme extraction, stakeholder categorization, urgency triage).
- Governance coordination: bias detection, escalation routing, traceability/audit trail generation.

### 1.1 Trigger & Configuration
Runs on a schedule and sets central configuration variables (API endpoints + thresholds) used throughout the workflow.

### 1.2 Data Retrieval & Submission Fan-out
Fetches consultation data from an API, then splits an array of submissions into per-submission items.

### 1.3 Per-submission AI Analysis (Claude) + Structured Parsing
Each submission is analyzed by a “Consultation Analysis Agent” (Claude), returning a structured JSON object validated by a structured output parser.

### 1.4 Aggregation + Governance Coordination (Claude + Tools)
All per-submission results are aggregated; a second agent (“Decision Governance Agent”) coordinates:
- Bias detection (via Code Tool)
- Executive summary generation (via Code Tool)
- Traceability record creation (via Code Tool)
Then returns a structured governance decision.

### 1.5 Bias Flagging, Escalation Routing & Record Consolidation
If bias is detected, it flags the record; then routes to critical/standard/routine preparation nodes; merges all paths and creates consolidated traceability records (prepared for storage).

---

## 2. Block-by-Block Analysis

### Block 1 — Trigger & Configuration
**Overview:** Starts executions on a schedule and defines workflow-wide configuration values (API URLs and thresholds).  
**Nodes involved:** `Schedule Trigger`, `Workflow Configuration`

#### Node: Schedule Trigger
- **Type / role:** `n8n-nodes-base.scheduleTrigger` — workflow entry point (time-based).
- **Key configuration:** Uses an interval rule with default/empty interval object in JSON (likely intended to be configured in UI).
- **Outputs:** To `Workflow Configuration`.
- **Edge cases / failures:**
  - Misconfigured schedule interval may result in no runs or unexpected frequency.
  - Timezone considerations depend on n8n instance settings.
- **Version:** 1.3

#### Node: Workflow Configuration
- **Type / role:** `n8n-nodes-base.set` — centralizes parameters for downstream expressions.
- **Key fields set:**
  - `consultationApiUrl` (placeholder)
  - `biasThreshold` = 0.7
  - `criticalEscalationThreshold` = 0.8
  - `standardReportThreshold` = 0.5
  - `traceabilityStorageUrl` (placeholder)
- **Important behavior:** `includeOtherFields: true` keeps incoming fields (from trigger) alongside config.
- **Outputs:** To `Fetch Consultation Data`.
- **Edge cases / failures:**
  - Placeholders not replaced → HTTP request will fail.
  - Thresholds are referenced in agent system prompts; if changed, they alter governance outcomes.
- **Version:** 3.4

---

### Block 2 — Data Retrieval & Submission Fan-out
**Overview:** Pulls consultation payload from an external API and splits `submissions[]` into individual items for analysis.  
**Nodes involved:** `Fetch Consultation Data`, `Split Submissions`

#### Node: Fetch Consultation Data
- **Type / role:** `n8n-nodes-base.httpRequest` — retrieves consultation data.
- **Configuration choices:**
  - `url` is an expression: `{{ $('Workflow Configuration').first().json.consultationApiUrl }}`
  - Sends header `Content-Type: application/json`
  - `sendHeaders: true`
- **Inputs:** From `Workflow Configuration`.
- **Outputs:** To `Split Submissions`.
- **Edge cases / failures:**
  - Auth not configured (no auth parameters here) → 401/403 if the API requires credentials.
  - Unexpected response shape (missing `submissions`) breaks the next node.
  - Timeouts / rate limits / pagination not handled (single call).
- **Version:** 4.3

#### Node: Split Submissions
- **Type / role:** `n8n-nodes-base.splitOut` — transforms an array into multiple items.
- **Configuration choices:**
  - `fieldToSplitOut: submissions`
  - `include: allOtherFields` ensures the parent fields are carried into each split item.
- **Inputs:** From `Fetch Consultation Data`.
- **Outputs:** To `Consultation Analysis Agent` (per submission).
- **Edge cases / failures:**
  - `submissions` missing or not an array → node errors or outputs zero items.
- **Version:** 1

---

### Block 3 — Per-submission AI Analysis (Claude) + Structured Parsing
**Overview:** Uses Claude to analyze each submission text and produce a structured analysis object enforced by a JSON-schema example.  
**Nodes involved:** `Anthropic Model - Analysis`, `Analysis Output Parser`, `Consultation Analysis Agent`, `Aggregate Analysis Results`

#### Node: Anthropic Model - Analysis
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatAnthropic` — provides Claude chat model to the agent.
- **Configuration choices:**
  - Model: `claude-sonnet-4-5-20250929` (cached name “Claude Sonnet 4.5”)
- **Credentials:** `Anthropic account` (Anthropic API key).
- **Connections:** Feeds as `ai_languageModel` into `Consultation Analysis Agent`.
- **Edge cases / failures:**
  - Invalid/expired Anthropic credential → auth errors.
  - Model availability / name mismatch across environments.
- **Version:** 1.3

#### Node: Analysis Output Parser
- **Type / role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — validates/structures the agent output.
- **Configuration choices:** Provides a JSON schema example with fields like:
  - `sentiment`, `keyThemes[]`, `stakeholderType`, `urgencyLevel`, `policyRelevance (0..1)`, etc.
- **Connections:** Connected to the agent as `ai_outputParser`.
- **Edge cases / failures:**
  - If the model output does not match the expected structure, parsing can fail and stop execution for that item.
  - Numeric constraints are described in text (“between 0 and 1”) but enforcement depends on parser behavior; malformed values can still occur.
- **Version:** 1.3

#### Node: Consultation Analysis Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — analyzes a single submission into structured fields.
- **Prompting configuration:**
  - `text`: `{{ $json.submissionText || $json.content }}`
  - System message enforces **objective extraction** and explicitly forbids policy recommendations.
  - Includes metadata placeholders: `submissionId`, `submitterType`, `submissionDate`, `location`.
  - `hasOutputParser: true` and instructs to “Return your analysis in the structured format defined by the output parser.”
- **Inputs:** From `Split Submissions` (one item per submission).
- **AI connections:**
  - Language model: `Anthropic Model - Analysis`
  - Output parser: `Analysis Output Parser`
- **Outputs:** To `Aggregate Analysis Results`.
- **Edge cases / failures:**
  - Missing `submissionText` and `content` → empty text; model may output low-quality or parser-failing output.
  - Long submissions may hit model context limits.
  - Per-item failures can halt the workflow unless “Continue On Fail” is enabled (not shown here).
- **Version:** 3.1

#### Node: Aggregate Analysis Results
- **Type / role:** `n8n-nodes-base.aggregate` — aggregates all item JSON into one object for governance.
- **Configuration choices:** `aggregate: aggregateAllItemData` (collects all items).
- **Inputs:** From `Consultation Analysis Agent`.
- **Outputs:** To `Decision Governance Agent`.
- **Edge cases / failures:**
  - If there are zero submissions, aggregated data may be empty; downstream tools assume `data.length` and can divide by zero (see summary tool).
- **Version:** 1

---

### Block 4 — Governance Coordination (Claude + Tools) + Structured Parsing
**Overview:** The governance agent uses Claude plus three code tools (bias detection, summary generation, traceability record generation) to decide escalation level and governance actions, then returns a structured governance decision.  
**Nodes involved:** `Anthropic Model - Governance`, `Bias Detection Tool`, `Summary Generation Tool`, `Traceability Record Tool`, `Governance Output Parser`, `Decision Governance Agent`

#### Node: Anthropic Model - Governance
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatAnthropic` — Claude model for governance agent.
- **Configuration:** Same model `claude-sonnet-4-5-20250929`.
- **Credentials:** `Anthropic account`.
- **Connections:** Provides `ai_languageModel` to `Decision Governance Agent`.
- **Edge cases:** Same as analysis model (auth/model availability).
- **Version:** 1.3

#### Node: Bias Detection Tool
- **Type / role:** `@n8n/n8n-nodes-langchain.toolCode` — JavaScript tool invoked by the agent.
- **Functionality (interpreted):**
  - Parses `query` JSON, expects `analysisData.data` (array of per-submission analyses) or uses the root value as array.
  - Computes several bias categories:
    - “demographic” (actually stakeholder type imbalance)
    - geographic scope imbalance
    - “stakeholderBias” (based on urgency/evidence quality proportions)
    - temporal clustering (if `submissionDate` exists)
  - Produces `biasScore` (weighted sum) and `biasDetected` if `biasScore > 0.4`.
- **Connections:** Available as `ai_tool` for `Decision Governance Agent`.
- **Edge cases / failures:**
  - If `data` is undefined/not an array → `data.forEach` throws.
  - `calculateVariance()` divides by mean; if mean is 0 (unlikely but possible for empty distributions) can produce `Infinity/NaN`.
  - Temporal parsing with invalid dates can yield `Invalid Date`.
- **Version:** 1.3

#### Node: Summary Generation Tool
- **Type / role:** `toolCode` — generates an executive summary string from aggregated analysis data.
- **Functionality:**
  - Counts sentiments/urgency/stakeholders, extracts top 5 themes, computes average policy relevance.
  - Returns a formatted text summary.
- **Connections:** Available as `ai_tool` for `Decision Governance Agent`.
- **Edge cases / failures:**
  - If `totalSubmissions` is 0, percentage calculations do `(v/0)` → `Infinity`/`NaN` strings.
  - Assumes `analysisData.data` exists; otherwise `totalSubmissions` is 0 and other maps stay empty.
- **Version:** 1.3

#### Node: Traceability Record Tool
- **Type / role:** `toolCode` — generates a traceability/audit JSON record.
- **Functionality:**
  - Parses governance metadata from `query`
  - Builds an audit record including:
    - decision points (bias, escalation, confidence)
    - data lineage (hardcoded node names)
    - governance actions
    - audit trail with `$execution.id`, `$execution.mode`, workflow name
  - Returns JSON string.
- **Connections:** Available as `ai_tool` for `Decision Governance Agent`.
- **Edge cases / failures:**
  - If `query` isn’t valid JSON string, parsing fails.
  - Hardcoded workflow name differs from actual name (“Public Consultation Analysis and Governance Coordination System” vs current workflow name “Smart … System”)—not a runtime failure, but an audit inconsistency.
- **Version:** 1.3

#### Node: Governance Output Parser
- **Type / role:** `outputParserStructured` — enforces final governance decision structure.
- **Schema example includes:** `governanceDecision`, `escalationLevel`, `biasDetected`, `biasScore`, `summaryGenerated`, `traceabilityRecordCreated`, `recommendedActions[]`, `stakeholdersToNotify[]`, `timelineForResponse`, `confidenceScore`, etc.
- **Connections:** As `ai_outputParser` to `Decision Governance Agent`.
- **Edge cases:** Model output mismatch → parsing failure.
- **Version:** 1.3

#### Node: Decision Governance Agent
- **Type / role:** `agent` — orchestrates governance steps and returns structured decision.
- **Prompting configuration:**
  - Input `text`: `{{ JSON.stringify($json) }}` (the aggregated analysis object)
  - System message:
    - prohibits policy recommendations
    - mandates tool call order: Bias Tool → Summary Tool → Traceability Tool → decide escalation/actions
    - includes escalation criteria and references configuration thresholds via:
      - `{{ $('Workflow Configuration').first().json.biasThreshold }}`
      - `{{ $('Workflow Configuration').first().json.criticalEscalationThreshold }}`
      - `{{ $('Workflow Configuration').first().json.standardReportThreshold }}`
  - `hasOutputParser: true`
- **Inputs:** From `Aggregate Analysis Results`.
- **AI connections:** Model `Anthropic Model - Governance`; tools: Bias/Summary/Traceability; parser: Governance Output Parser.
- **Outputs:** To `Check for Bias Flags`.
- **Edge cases / failures:**
  - If aggregated object is large, `JSON.stringify($json)` may exceed context length.
  - The agent’s internal use of thresholds is “soft” (prompt-based); the bias tool itself uses a fixed 0.4 detection threshold, not `biasThreshold`.
- **Version:** 3.1

---

### Block 5 — Bias Flagging, Escalation Routing & Consolidation
**Overview:** Adds a bias alert flag when bias is detected, routes outputs by escalation level, prepares notification metadata, merges paths, and creates consolidated traceability records (prepared for external storage).  
**Nodes involved:** `Check for Bias Flags`, `Flag Bias Alert`, `Route by Escalation Level`, `Prepare Critical Escalation`, `Prepare Standard Report`, `Prepare Routine Update`, `Merge Escalation Paths`, `Store Traceability Records`

#### Node: Check for Bias Flags
- **Type / role:** `n8n-nodes-base.if` — conditional branch on bias detection.
- **Condition:** Checks `{{ $('Decision Governance Agent').item.json.biasDetected }} == true`
  - Note: It references the agent node directly, not `$json.biasDetected`.
- **Outputs:**
  - **True branch:** `Flag Bias Alert`
  - **False branch:** `Route by Escalation Level`
- **Edge cases / failures:**
  - Using `$('Decision Governance Agent').item` assumes item alignment; if multiple items or merges occur upstream unexpectedly, this can mismatch.
  - If the governance agent fails and doesn’t output `biasDetected`, condition may evaluate false or error depending on n8n settings.
- **Version:** 2.3

#### Node: Flag Bias Alert
- **Type / role:** `set` — annotates item with bias alert metadata.
- **Fields set:**
  - `biasAlertFlag: true`
  - `biasScore: {{ $json.biasScore }}`
  - `biasDetails` stored as **stringified JSON**: `{{ JSON.stringify($json.biasDetails) }}`
  - `alertTimestamp: {{ $now.toISO() }}`
  - `requiresReview: true`
- **Outputs:** To `Route by Escalation Level`.
- **Edge cases:**
  - Converting `biasDetails` to string may complicate later consumers expecting an array.
- **Version:** 3.4

#### Node: Route by Escalation Level
- **Type / role:** `switch` — routes to one of three paths based on `escalationLevel`.
- **Rules:**
  - If `{{ $json.escalationLevel }}` equals `critical` → output “Critical”
  - equals `standard` → “Standard”
  - equals `routine` → “Routine”
- **Outputs:**
  - Critical → `Prepare Critical Escalation`
  - Standard → `Prepare Standard Report`
  - Routine → `Prepare Routine Update`
- **Edge cases / failures:**
  - If `escalationLevel` is missing or outside these values, **no output** is produced (item dropped unless fallback configured).
- **Version:** 3.4

#### Node: Prepare Critical Escalation
- **Type / role:** `set` — builds notification payload for critical cases.
- **Fields set:** `escalationType=critical`, `notificationPriority=immediate`, `recipientGroup=senior_policymakers`, subject/body composed via expressions, `escalationTimestamp`.
- **Outputs:** To `Merge Escalation Paths`.
- **Edge cases:** Uses `JSON.stringify($json.recommendedActions)`; if already string, will double-quote/escape.
- **Version:** 3.4

#### Node: Prepare Standard Report
- **Type / role:** `set` — builds payload for standard review.
- **Fields set:** `escalationType=standard`, `notificationPriority=24_hours`, `recipientGroup=policy_team`, subject/body, `reportTimestamp`.
- **Outputs:** To `Merge Escalation Paths`.
- **Version:** 3.4

#### Node: Prepare Routine Update
- **Type / role:** `set` — builds payload for routine update.
- **Fields set:** `escalationType=routine`, `notificationPriority=7_days`, `recipientGroup=consultation_team`, subject/body, `updateTimestamp`.
- **Outputs:** To `Merge Escalation Paths`.
- **Version:** 3.4

#### Node: Merge Escalation Paths
- **Type / role:** `n8n-nodes-base.merge` — merges up to three inputs.
- **Configuration:** `numberInputs: 3`
- **Inputs:** From the three “Prepare …” nodes.
- **Outputs:** To `Store Traceability Records`.
- **Edge cases:**
  - Merge behavior depends on merge mode (not specified in parameters); with multiple active branches, ensure expected item pairing/concatenation.
- **Version:** 3.2

#### Node: Store Traceability Records
- **Type / role:** `n8n-nodes-base.code` — consolidates escalation outputs into audit records and prepares a storage payload.
- **Functionality:**
  - Iterates through `$input.all()` and creates a `traceabilityRecords[]` array with:
    - escalation metadata, governance decision details, bias results, notification tracking, and a `fullDataSnapshot`
  - Produces `consolidatedOutput` including summary stats:
    - counts by escalation type, bias alerts triggered, average bias/confidence
  - Reads `traceabilityStorageUrl` from configuration but **does not actually send**; sets `storageStatus: 'pending_api_call'`.
- **Outputs:** Final node (no downstream connections).
- **Edge cases / failures:**
  - If no items arrive, averages divide by `traceabilityRecords.length` (0) → `NaN`.
  - Record IDs created here are separate from IDs from Traceability Record Tool; duplication/inconsistency possible.
- **Version:** 2

---

### Sticky Notes (documentation-only nodes)
The workflow contains multiple sticky notes; they do not execute but provide context. Several mention ESG/compliance concepts that do **not** match the public consultation logic—treat them as reused board notes.

- `Sticky Note`: “Automated Data Collection & Dual-Agent Validation…”
- `Sticky Note1`: “Prerequisites / Use Cases / Customization / Benefits…”
- `Sticky Note2`: “Setup Steps…”
- `Sticky Note3`: “How It Works…” (ESG-focused narrative)
- `Sticky Note4`: “Quality Rating Assessment…”
- `Sticky Note5`: “Intelligent Report Generation…”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | Schedule Trigger | Time-based entry point | — | Workflow Configuration | ## Automated Data Collection & Dual-Agent Validation<br>**What**: Compliance Analyzer validates regulatory requirements; Decision Agent coordinates specialized ESG sub-agents<br>**Why**: Parallel compliance checking and data quality analysis ensures accuracy across multiple reporting frameworks |
| Workflow Configuration | Set | Define API endpoints + thresholds | Schedule Trigger | Fetch Consultation Data | ## Automated Data Collection & Dual-Agent Validation<br>**What**: Compliance Analyzer validates regulatory requirements; Decision Agent coordinates specialized ESG sub-agents<br>**Why**: Parallel compliance checking and data quality analysis ensures accuracy across multiple reporting frameworks |
| Fetch Consultation Data | HTTP Request | Pull consultation payload | Workflow Configuration | Split Submissions | ## Automated Data Collection & Dual-Agent Validation<br>**What**: Compliance Analyzer validates regulatory requirements; Decision Agent coordinates specialized ESG sub-agents<br>**Why**: Parallel compliance checking and data quality analysis ensures accuracy across multiple reporting frameworks |
| Split Submissions | Split Out | Fan-out submissions array | Fetch Consultation Data | Consultation Analysis Agent | ## Automated Data Collection & Dual-Agent Validation<br>**What**: Compliance Analyzer validates regulatory requirements; Decision Agent coordinates specialized ESG sub-agents<br>**Why**: Parallel compliance checking and data quality analysis ensures accuracy across multiple reporting frameworks |
| Anthropic Model - Analysis | LangChain Anthropic Chat Model | Claude model for submission analysis | — (AI connection) | Consultation Analysis Agent (ai_languageModel) | ## Automated Data Collection & Dual-Agent Validation<br>**What**: Compliance Analyzer validates regulatory requirements; Decision Agent coordinates specialized ESG sub-agents<br>**Why**: Parallel compliance checking and data quality analysis ensures accuracy across multiple reporting frameworks |
| Analysis Output Parser | Structured Output Parser | Enforce submission-analysis schema | — (AI connection) | Consultation Analysis Agent (ai_outputParser) | ## Automated Data Collection & Dual-Agent Validation<br>**What**: Compliance Analyzer validates regulatory requirements; Decision Agent coordinates specialized ESG sub-agents<br>**Why**: Parallel compliance checking and data quality analysis ensures accuracy across multiple reporting frameworks |
| Consultation Analysis Agent | LangChain Agent | Per-submission categorization/extraction | Split Submissions | Aggregate Analysis Results | ## Automated Data Collection & Dual-Agent Validation<br>**What**: Compliance Analyzer validates regulatory requirements; Decision Agent coordinates specialized ESG sub-agents<br>**Why**: Parallel compliance checking and data quality analysis ensures accuracy across multiple reporting frameworks |
| Aggregate Analysis Results | Aggregate | Combine all analyses for governance | Consultation Analysis Agent | Decision Governance Agent | ## Quality Rating Assessment<br>**What**: Checks ESG data completeness and reliability scores against established quality thresholds<br>**Why**: Quality-based filtering identifies data gaps requiring remediation before regulatory submission deadlines |
| Anthropic Model - Governance | LangChain Anthropic Chat Model | Claude model for governance agent | — (AI connection) | Decision Governance Agent (ai_languageModel) | ## Quality Rating Assessment<br>**What**: Checks ESG data completeness and reliability scores against established quality thresholds<br>**Why**: Quality-based filtering identifies data gaps requiring remediation before regulatory submission deadlines |
| Bias Detection Tool | LangChain Code Tool | Compute bias metrics/flags | — (AI tool) | Decision Governance Agent (ai_tool) | ## Quality Rating Assessment<br>**What**: Checks ESG data completeness and reliability scores against established quality thresholds<br>**Why**: Quality-based filtering identifies data gaps requiring remediation before regulatory submission deadlines |
| Summary Generation Tool | LangChain Code Tool | Generate executive summary text | — (AI tool) | Decision Governance Agent (ai_tool) | ## Quality Rating Assessment<br>**What**: Checks ESG data completeness and reliability scores against established quality thresholds<br>**Why**: Quality-based filtering identifies data gaps requiring remediation before regulatory submission deadlines |
| Traceability Record Tool | LangChain Code Tool | Build audit/traceability JSON | — (AI tool) | Decision Governance Agent (ai_tool) | ## Quality Rating Assessment<br>**What**: Checks ESG data completeness and reliability scores against established quality thresholds<br>**Why**: Quality-based filtering identifies data gaps requiring remediation before regulatory submission deadlines |
| Governance Output Parser | Structured Output Parser | Enforce governance-decision schema | — (AI connection) | Decision Governance Agent (ai_outputParser) | ## Quality Rating Assessment<br>**What**: Checks ESG data completeness and reliability scores against established quality thresholds<br>**Why**: Quality-based filtering identifies data gaps requiring remediation before regulatory submission deadlines |
| Decision Governance Agent | LangChain Agent | Orchestrate governance + escalation decision | Aggregate Analysis Results | Check for Bias Flags | ## Quality Rating Assessment<br>**What**: Checks ESG data completeness and reliability scores against established quality thresholds<br>**Why**: Quality-based filtering identifies data gaps requiring remediation before regulatory submission deadlines |
| Check for Bias Flags | IF | Branch if bias detected | Decision Governance Agent | Flag Bias Alert; Route by Escalation Level | ## Intelligent Report Generation<br>**What**: Routes by status—critical issues trigger immediate alerts, routine generates standard reports with traceability<br>**Why**: Risk-based workflows ensure compliance violations receive urgent attention while automating routine disclosures |
| Flag Bias Alert | Set | Annotate payload with bias alert fields | Check for Bias Flags (true) | Route by Escalation Level | ## Intelligent Report Generation<br>**What**: Routes by status—critical issues trigger immediate alerts, routine generates standard reports with traceability<br>**Why**: Risk-based workflows ensure compliance violations receive urgent attention while automating routine disclosures |
| Route by Escalation Level | Switch | Route to critical/standard/routine prep | Check for Bias Flags (false) / Flag Bias Alert | Prepare Critical Escalation; Prepare Standard Report; Prepare Routine Update | ## Intelligent Report Generation<br>**What**: Routes by status—critical issues trigger immediate alerts, routine generates standard reports with traceability<br>**Why**: Risk-based workflows ensure compliance violations receive urgent attention while automating routine disclosures |
| Prepare Critical Escalation | Set | Build critical notification payload | Route by Escalation Level | Merge Escalation Paths | ## Intelligent Report Generation<br>**What**: Routes by status—critical issues trigger immediate alerts, routine generates standard reports with traceability<br>**Why**: Risk-based workflows ensure compliance violations receive urgent attention while automating routine disclosures |
| Prepare Standard Report | Set | Build standard review payload | Route by Escalation Level | Merge Escalation Paths | ## Intelligent Report Generation<br>**What**: Routes by status—critical issues trigger immediate alerts, routine generates standard reports with traceability<br>**Why**: Risk-based workflows ensure compliance violations receive urgent attention while automating routine disclosures |
| Prepare Routine Update | Set | Build routine update payload | Route by Escalation Level | Merge Escalation Paths | ## Intelligent Report Generation<br>**What**: Routes by status—critical issues trigger immediate alerts, routine generates standard reports with traceability<br>**Why**: Risk-based workflows ensure compliance violations receive urgent attention while automating routine disclosures |
| Merge Escalation Paths | Merge | Merge the three escalation outputs | Prepare Critical Escalation; Prepare Standard Report; Prepare Routine Update | Store Traceability Records | ## Intelligent Report Generation<br>**What**: Routes by status—critical issues trigger immediate alerts, routine generates standard reports with traceability<br>**Why**: Risk-based workflows ensure compliance violations receive urgent attention while automating routine disclosures |
| Store Traceability Records | Code | Consolidate and prepare audit storage payload | Merge Escalation Paths | — | ## Intelligent Report Generation<br>**What**: Routes by status—critical issues trigger immediate alerts, routine generates standard reports with traceability<br>**Why**: Risk-based workflows ensure compliance violations receive urgent attention while automating routine disclosures |
| Sticky Note | Sticky Note | Comment / canvas note | — | — |  |
| Sticky Note1 | Sticky Note | Comment / canvas note | — | — |  |
| Sticky Note2 | Sticky Note | Comment / canvas note | — | — |  |
| Sticky Note3 | Sticky Note | Comment / canvas note | — | — |  |
| Sticky Note4 | Sticky Note | Comment / canvas note | — | — |  |
| Sticky Note5 | Sticky Note | Comment / canvas note | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **Smart Public Consultation Analysis and Governance Coordination System**
   - Keep it inactive until credentials and endpoints are set.

2. **Add “Schedule Trigger”**
   - Node type: **Schedule Trigger**
   - Configure the desired interval (e.g., daily/weekly).  
   - This is the workflow entry point.

3. **Add “Workflow Configuration” (Set)**
   - Node type: **Set**
   - Enable **Include Other Fields**.
   - Add fields:
     - `consultationApiUrl` (string) → your consultation API endpoint
     - `biasThreshold` (number) → 0.7
     - `criticalEscalationThreshold` (number) → 0.8
     - `standardReportThreshold` (number) → 0.5
     - `traceabilityStorageUrl` (string) → your storage endpoint (optional for now)
   - Connect: **Schedule Trigger → Workflow Configuration**

4. **Add “Fetch Consultation Data” (HTTP Request)**
   - Node type: **HTTP Request**
   - URL (expression): `{{ $('Workflow Configuration').first().json.consultationApiUrl }}`
   - Method: GET (default) unless your API requires otherwise
   - Headers: `Content-Type: application/json`
   - Configure authentication if your API requires it (Bearer, OAuth2, etc.).
   - Connect: **Workflow Configuration → Fetch Consultation Data**

5. **Add “Split Submissions”**
   - Node type: **Split Out**
   - Field to split out: `submissions`
   - Include: **All Other Fields**
   - Connect: **Fetch Consultation Data → Split Submissions**

6. **Add “Anthropic Model - Analysis”**
   - Node type: **LangChain > Anthropic Chat Model**
   - Model: `claude-sonnet-4-5-20250929` (or the closest available equivalent in your environment)
   - Credentials: create/select **Anthropic API** credentials (API key).
   - No main connection needed (AI link used by the agent).

7. **Add “Analysis Output Parser”**
   - Node type: **LangChain > Structured Output Parser**
   - Paste the schema example used in the workflow (fields for sentiment, themes, urgency, policyRelevance, etc.).

8. **Add “Consultation Analysis Agent”**
   - Node type: **LangChain > Agent**
   - Prompt mode: “Define”
   - Text expression: `{{ $json.submissionText || $json.content }}`
   - System message: replicate the constraints (objective extraction; no policy recommendations) and instruct it to output per the parser format.
   - Attach AI components:
     - **Language Model:** connect **Anthropic Model - Analysis** to the agent’s **ai_languageModel**
     - **Output Parser:** connect **Analysis Output Parser** to **ai_outputParser**
   - Connect main flow: **Split Submissions → Consultation Analysis Agent**

9. **Add “Aggregate Analysis Results”**
   - Node type: **Aggregate**
   - Mode: **Aggregate all item data**
   - Connect: **Consultation Analysis Agent → Aggregate Analysis Results**

10. **Add governance AI components**
   - **Anthropic Model - Governance**
     - Node type: Anthropic Chat Model
     - Same credential + model selection as above
   - **Bias Detection Tool**
     - Node type: LangChain **Code Tool**
     - Paste the bias detection JS code (as in workflow)
   - **Summary Generation Tool**
     - Node type: LangChain **Code Tool**
     - Paste summary generation JS code
   - **Traceability Record Tool**
     - Node type: LangChain **Code Tool**
     - Paste traceability record JS code
   - **Governance Output Parser**
     - Node type: Structured Output Parser
     - Paste governance decision schema example

11. **Add “Decision Governance Agent”**
   - Node type: **LangChain > Agent**
   - Text expression: `{{ JSON.stringify($json) }}`
   - System message: replicate governance coordination instructions, escalation criteria, and threshold references:
     - `{{ $('Workflow Configuration').first().json.biasThreshold }}`
     - `{{ $('Workflow Configuration').first().json.criticalEscalationThreshold }}`
     - `{{ $('Workflow Configuration').first().json.standardReportThreshold }}`
   - Attach:
     - **Language Model:** `Anthropic Model - Governance` → agent `ai_languageModel`
     - **Tools:** connect each tool node to the agent `ai_tool`
     - **Output Parser:** `Governance Output Parser` → agent `ai_outputParser`
   - Connect main flow: **Aggregate Analysis Results → Decision Governance Agent**

12. **Add “Check for Bias Flags” (IF)**
   - Node type: **IF**
   - Condition (boolean equals true):
     - Left value: `{{ $('Decision Governance Agent').item.json.biasDetected }}`
     - Right value: `true`
   - Connect: **Decision Governance Agent → Check for Bias Flags**

13. **Add “Flag Bias Alert” (Set)**
   - Node type: **Set**
   - Include other fields: true
   - Add:
     - `biasAlertFlag` = true
     - `biasScore` = `{{ $json.biasScore }}`
     - `biasDetails` = `{{ JSON.stringify($json.biasDetails) }}`
     - `alertTimestamp` = `{{ $now.toISO() }}`
     - `requiresReview` = true
   - Connect IF true: **Check for Bias Flags (true) → Flag Bias Alert**

14. **Add “Route by Escalation Level” (Switch)**
   - Node type: **Switch**
   - Create 3 rules:
     - `{{ $json.escalationLevel }}` equals `critical`
     - equals `standard`
     - equals `routine`
   - Connect:
     - IF false: **Check for Bias Flags (false) → Route by Escalation Level**
     - From bias path: **Flag Bias Alert → Route by Escalation Level**

15. **Add the three “Prepare …” Set nodes**
   - **Prepare Critical Escalation**
     - Fields: `escalationType=critical`, `notificationPriority=immediate`, `recipientGroup=senior_policymakers`, subject/body, timestamp `{{ $now.toISO() }}`
   - **Prepare Standard Report**
     - Fields: `escalationType=standard`, `notificationPriority=24_hours`, `recipientGroup=policy_team`, subject/body, timestamp
   - **Prepare Routine Update**
     - Fields: `escalationType=routine`, `notificationPriority=7_days`, `recipientGroup=consultation_team`, subject/body, timestamp
   - Connect Switch outputs:
     - Critical → Prepare Critical Escalation
     - Standard → Prepare Standard Report
     - Routine → Prepare Routine Update

16. **Add “Merge Escalation Paths”**
   - Node type: **Merge**
   - Number of inputs: **3**
   - Connect:
     - Prepare Critical Escalation → Merge (input 1)
     - Prepare Standard Report → Merge (input 2)
     - Prepare Routine Update → Merge (input 3)

17. **Add “Store Traceability Records” (Code)**
   - Node type: **Code**
   - Paste the consolidation JS (builds `traceabilityRecords`, summary stats, and references `traceabilityStorageUrl`).
   - Connect: **Merge Escalation Paths → Store Traceability Records**
   - (Optional) Add an **HTTP Request** node afterwards to POST `consolidatedOutput` to `traceabilityStorageUrl`.

18. **Credentials**
   - Create **Anthropic API** credentials in n8n and select them in both Anthropic model nodes.
   - Configure consultation API authentication in `Fetch Consultation Data` if needed.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Sticky notes describe ESG/compliance “dual-agent validation” and “quality rating assessment”, which appear reused and do not fully match the public consultation logic in the executable nodes. | Canvas notes: “How It Works”, “Prerequisites”, “Setup Steps”, “Quality Rating Assessment”, “Intelligent Report Generation”. |
| The final node prepares traceability records but does not actually store them; it explicitly indicates adding an HTTP Request to the storage API endpoint. | `Store Traceability Records` → `storageStatus: 'pending_api_call'` / “Implement HTTP Request to storage API endpoint.” |
| Bias detection tool uses a fixed detection threshold (`biasScore > 0.4`) which may differ from `biasThreshold` in configuration (0.7). Align if you need consistent governance rules. | Bias tool code vs Workflow Configuration + governance agent prompt. |
| Zero-submission cases can produce division-by-zero artifacts in summary and averaging; consider adding guards. | Summary Generation Tool; Store Traceability Records. |
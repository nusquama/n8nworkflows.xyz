Detect financial risk and orchestrate compliance with GPT‑4o, Slack and email

https://n8nworkflows.xyz/workflows/detect-financial-risk-and-orchestrate-compliance-with-gpt-4o--slack-and-email-13141


# Detect financial risk and orchestrate compliance with GPT‑4o, Slack and email

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title (provided):** Detect financial risk and orchestrate compliance with GPT‑4o, Slack and email  
**Workflow name (JSON):** Smart Risk Assessment and Compliance Orchestration Automation System

**Purpose:**  
On a scheduled basis, the workflow fetches *risk/exposure* data and *claims* data from two APIs, merges and enriches them with computed metrics, asks GPT‑4o to produce a structured risk assessment (score + level + factors), routes the case based on severity, orchestrates compliance actions via another GPT‑4o agent (with regulatory + exception sub-tools), sends notifications (Slack for critical; email report), and appends/updates a Google Sheets audit trail with generated metadata.

### Logical blocks
1.1 **Scheduled ingestion & configuration** – schedule trigger + centralized configuration variables.  
1.2 **Multi-source data retrieval & consolidation** – fetch risk + claims via HTTP; merge datasets.  
1.3 **Risk metrics enrichment** – compute aggregate exposure/claims metrics and concentration/volatility signals.  
1.4 **AI risk assessment (GPT‑4o)** – risk agent produces JSON parsed into a strict schema.  
1.5 **Severity routing & escalation** – IF + Switch route to Slack escalation and/or compliance orchestration paths.  
1.6 **Compliance orchestration (GPT‑4o + tools)** – compliance agent coordinates actions, approvals, regulatory items, exceptions; structured output parsing.  
1.7 **Auditability & reporting** – audit metadata generation; Google Sheets logging; consolidated results; email report.

---

## 2. Block-by-Block Analysis

### 1.1 Scheduled ingestion & configuration
**Overview:** Triggers daily execution and defines all environment-specific parameters (API URLs, thresholds, Slack channel, audit sheet ID, email). Centralizes variables to avoid hardcoding across nodes.

**Nodes involved:**
- Schedule Trigger
- Workflow Configuration

**Node details**

**Schedule Trigger**
- **Type / role:** `scheduleTrigger` – entry point; time-based automation.
- **Configuration:** Runs at **09:00** (interval rule with `triggerAtHour: 9`).
- **Inputs/outputs:** No input; outputs to **Workflow Configuration**.
- **Edge cases:** Timezone depends on n8n instance settings; if you expect business-timezone behavior, ensure instance timezone is set accordingly.

**Workflow Configuration**
- **Type / role:** `set` – defines reusable constants for the run.
- **Configuration choices:**
  - Placeholder string values: `riskDataApiUrl`, `claimsDataApiUrl`, `complianceEmail`, `slackChannel`, `auditSheetId`, `fromEmail`.
  - Numeric thresholds: `riskThresholdHigh = 75`, `riskThresholdMedium = 50`, `criticalRiskThreshold = 90` (note: **criticalRiskThreshold is not used** in routing logic).
  - `includeOtherFields: true` keeps any upstream fields (none expected from trigger).
- **Key variables used later:**
  - `$('Workflow Configuration').first().json.riskDataApiUrl` (HTTP)
  - `...claimsDataApiUrl` (HTTP)
  - `...complianceEmail` (Email)
  - `...slackChannel` (Slack)
  - `...auditSheetId` (Sheets)
  - `...riskThresholdHigh` (IF node; also duplicated in Switch as a literal 75)
- **Outputs:** Fan-out to **Fetch Risk Data** and **Fetch Claims Data**.
- **Edge cases:** Placeholders must be replaced; otherwise HTTP nodes will fail with invalid URL and email/slack/sheets will fail or send nowhere.

---

### 1.2 Multi-source data retrieval & consolidation
**Overview:** Pulls two datasets (risk/exposure and claims) over HTTP and merges them into a combined stream for analysis.

**Nodes involved:**
- Fetch Risk Data
- Fetch Claims Data
- Merge Financial Data

**Node details**

**Fetch Risk Data**
- **Type / role:** `httpRequest` – retrieves risk/exposure JSON from external API.
- **Configuration:**
  - URL: expression from configuration: `={{ $('Workflow Configuration').first().json.riskDataApiUrl }}`
  - Sends header `Content-Type: application/json`
  - Default method is GET unless API requires otherwise.
- **Outputs:** To **Merge Financial Data** input index 0.
- **Failure types / edge cases:** DNS/network issues, 401/403 (missing auth), 429 rate limits, non-JSON response.

**Fetch Claims Data**
- **Type / role:** `httpRequest` – retrieves claims JSON from external API.
- **Configuration:** Same pattern as above; URL uses `claimsDataApiUrl`.
- **Outputs:** To **Merge Financial Data** input index 1.
- **Failure types / edge cases:** Same as above; plus schema differences across claims endpoints.

**Merge Financial Data**
- **Type / role:** `merge` – combines two streams by position.
- **Configuration:** `mode: combine`, `combineByPosition`.
- **Outputs:** To **Risk Signal Agent** and **Calculate Risk Metrics**.
- **Edge cases:**
  - If one HTTP call returns multiple items and the other returns a different count, “combine by position” can misalign risk and claims records.
  - If an endpoint returns a single object vs. array, you may end up with unexpected item structure.

---

### 1.3 Risk metrics enrichment
**Overview:** Computes advanced aggregate metrics (HHI concentration indices, volatility, loss ratio, trends) from merged items and forwards enriched data into the AI risk assessment.

**Nodes involved:**
- Calculate Risk Metrics

**Node details**

**Calculate Risk Metrics**
- **Type / role:** `code` – JS computation of exposure/claims analytics.
- **Configuration choices (interpreted):**
  - Aggregates:
    - `totalExposure` from `data.exposure.amount`
    - `totalClaims` and `claimsHistory` from `data.claims`
  - Concentrations by geography/industry/counterparty, then computes **HHI** scaled to 0–10000.
  - Claims volatility: standard deviation of claim amounts.
  - Loss ratio: `totalClaims / totalExposure * 100`.
  - Trend heuristic: compares last ~3 values vs earlier values.
  - Outputs a single item with:
    - `riskMetrics`, `concentrationIndices`, `topConcentrations`, `claimsHistory`, `timestamp`.
- **Inputs/outputs:** Input from **Merge Financial Data**; outputs to **Risk Signal Agent**.
- **Edge cases / failure types:**
  - If `totalExposure` is 0, HHI calculation divides by 0 (`share = value/total`) producing `Infinity/NaN`.
  - If merged payload does not contain `exposure`/`claims` keys, many values will be 0 or “Unknown”.
  - `lossRatio` is returned as a string (`toFixed(2)`), while downstream AI schema expects `claimsTrends.lossRatio` as a number (in the *risk agent output*, not in this node). Be mindful of numeric typing if you later reuse `riskMetrics.lossRatio`.

---

### 1.4 AI risk assessment (GPT‑4o)
**Overview:** GPT‑4o analyzes structured merged+computed data and returns a strict JSON risk assessment which is parsed and used for routing and reporting.

**Nodes involved:**
- Risk Signal Agent
- OpenAI Model - Risk Agent
- Risk Assessment Output Parser

**Node details**

**Risk Signal Agent**
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` – orchestrates a chat-based agent with instructions and structured output.
- **Configuration choices:**
  - Prompt text: `Risk Data: {{ JSON.stringify($json) }}` (sends current merged+metrics item).
  - System message: defines responsibilities and constraints (analytical only; no underwriting/approval).
  - `hasOutputParser: true` – uses the connected structured parser.
- **Connections:**
  - AI Language Model: **OpenAI Model - Risk Agent**
  - Output Parser: **Risk Assessment Output Parser**
  - Main outputs: to **Route by Risk Level** and **Check Critical Risk**
- **Edge cases:**
  - JSON string may be very large (token pressure) if APIs return big arrays.
  - If the model outputs invalid JSON, parsing fails (node error).

**OpenAI Model - Risk Agent**
- **Type / role:** `lmChatOpenAi` – GPT‑4o backend for the agent.
- **Configuration:** `model: gpt-4o`, `temperature: 0.2`.
- **Credentials:** OpenAI account credential in n8n.
- **Failure types:** Invalid API key, quota/rate limits, model availability, request too large.

**Risk Assessment Output Parser**
- **Type / role:** `outputParserStructured` – validates model output against a JSON Schema.
- **Schema requirements:**
  - Required: `riskScore` (number), `riskLevel` (string), `keyRiskFactors` (string[]), `reasoning` (string).
  - Optional: `exposureConcentration` object, `claimsTrends` object, etc.
- **Edge cases:**
  - If the model returns `riskScore` as a string, strict schema may reject depending on parser behavior/version.
  - Missing required keys causes parsing failure and stops downstream routing.

---

### 1.5 Severity routing & escalation
**Overview:** Determines whether risk is at/above “high” threshold and routes into (a) critical Slack escalation path and (b) severity buckets for orchestration and reporting.

**Nodes involved:**
- Check Critical Risk
- Send Critical Alert to Slack
- Merge Critical Path
- Route by Risk Level

**Node details**

**Check Critical Risk**
- **Type / role:** `if` – gate for “critical/high” escalation.
- **Configuration:**
  - Condition: `riskScore >= $('Workflow Configuration').first().json.riskThresholdHigh`
  - Despite node name, it uses **riskThresholdHigh (75)**, not `criticalRiskThreshold (90)`.
- **Outputs:**
  - True → **Send Critical Alert to Slack**
  - False → **Route by Risk Level**
- **Edge cases:** If `$json.riskScore` is undefined (parser failure or wrong field), comparison can behave unexpectedly (loose validation enabled).

**Send Critical Alert to Slack**
- **Type / role:** `slack` – sends multi-line critical message.
- **Configuration:**
  - OAuth2 authentication
  - Channel selected by ID from config: `={{ $('Workflow Configuration').first().json.slackChannel }}`
  - Message uses `$json.output.*` (agent output container) for risk score/level/factors and optional fields via `?.`.
- **Outputs:** To **Merge Critical Path**.
- **Failure types:** Slack OAuth token expired, missing channel access, invalid channel ID, message formatting errors (rare).

**Merge Critical Path**
- **Type / role:** `merge` – recombines the “critical alert sent” path with the severity-routing path before compliance orchestration.
- **Configuration:** `combineByPosition`.
- **Inputs:**
  - Input 0: from **Send Critical Alert to Slack**
  - Input 1: from **Route by Risk Level** (High Risk output)
- **Output:** To **Compliance Orchestration Agent**.
- **Edge cases:** If only one side produces items (e.g., slack step errors or route mismatch), combine-by-position can stall or miscombine.

**Route by Risk Level**
- **Type / role:** `switch` – buckets risk score into High / Medium / Low.
- **Configuration rules:**
  - “High Risk”: `riskScore >= 75` (hardcoded)
  - “Medium Risk”: `50 <= riskScore < 75` (hardcoded)
  - Fallback output renamed “Low Risk”
- **Outputs:**
  - High Risk → **Compliance Orchestration Agent** *and* **Merge Critical Path** (second connection)
  - Medium Risk → **Consolidate Results** (index 1)
  - Low Risk (fallback) has no outgoing connection in this workflow (implicit end).
- **Edge cases / design gaps:**
  - Hardcoded thresholds may drift from `Workflow Configuration` values.
  - Low risk path is not documented/logged/sent anywhere; likely unintended.

---

### 1.6 Compliance orchestration (GPT‑4o + tools)
**Overview:** A second GPT‑4o agent produces compliance workflow actions, approvals, regulatory items, exceptions and audit packaging. It uses two agent “tools” (regulatory analysis and exception handling), each with their own GPT‑4o model and structured parsers.

**Nodes involved:**
- Compliance Orchestration Agent
- OpenAI Model - Compliance Agent
- Compliance Output Parser
- Regulatory Analysis Agent Tool
- OpenAI Model - Regulatory Tool
- Regulatory Tool Output Parser
- Exception Handler Agent Tool
- OpenAI Model - Exception Tool
- Exception Tool Output Parser

**Node details**

**Compliance Orchestration Agent**
- **Type / role:** `agent` – coordinates compliance actions and calls tools.
- **Configuration:**
  - Text: `Risk Assessment: {{ JSON.stringify($json.output) }}`
  - System message: enforces coordination-only role; references SOX/Basel III/Solvency II; requires audit-ready package and structured JSON.
  - `hasOutputParser: true` – connects to **Compliance Output Parser**.
  - Tools connected:
    - **Regulatory Analysis Agent Tool**
    - **Exception Handler Agent Tool**
- **Inputs/outputs:**
  - Inputs from **Merge Critical Path** (critical/high) and directly from **Route by Risk Level** (high).
  - Outputs to **Consolidate Results** and **Generate Audit Metadata**.
- **Edge cases:**
  - If `$json.output` is missing (upstream parsing changed), agent context becomes empty and output quality drops or parsing fails.
  - Tool invocation depends on agent behavior; if it doesn’t call tools, regulatory/exception detail may be thin.

**OpenAI Model - Compliance Agent**
- **Type / role:** GPT‑4o model for compliance agent.
- **Configuration:** `gpt-4o`, `temperature: 0.2`.
- **Failure types:** Same as other OpenAI nodes.

**Compliance Output Parser**
- **Type / role:** Structured schema validator for compliance agent output.
- **Required:** `workflowActions`, `requiredApprovals`, `regulatoryItems`, `auditPackage`, `summary`.
- **Optional:** `exceptionHandling` object (not required by schema but used in email template).
- **Edge cases:**
  - Email template later expects `exceptionHandling` fields; if missing, template expressions can error unless guarded (some are guarded, but not all).

**Regulatory Analysis Agent Tool**
- **Type / role:** `agentTool` – callable by the compliance agent for regulatory assessment.
- **Configuration:**
  - Tool input text: `={{ $fromAI('riskData', 'Risk assessment data requiring regulatory analysis', 'json') }}`
  - System message: regulatory framework identification, reporting obligations, capital adequacy, gaps and remediation.
  - Has output parser.
- **Model & parser:** Uses **OpenAI Model - Regulatory Tool** + **Regulatory Tool Output Parser**.
- **Edge cases:** `$fromAI(...)` requires correct agent-tool wiring; if the agent does not provide `riskData` as expected, tool input may be empty.

**OpenAI Model - Regulatory Tool**
- **Type / role:** GPT‑4o for regulatory tool.
- **Configuration:** `temperature: 0.1` (more deterministic).

**Regulatory Tool Output Parser**
- **Type / role:** Validates tool output.
- **Required:** `applicableFrameworks`, `complianceRequirements`.

**Exception Handler Agent Tool**
- **Type / role:** `agentTool` – callable exception analysis.
- **Configuration:** Similar `$fromAI('riskData', ...)` pattern; outputs structured exception analysis.
- **Model & parser:** **OpenAI Model - Exception Tool** + **Exception Tool Output Parser**.

**OpenAI Model - Exception Tool**
- **Type / role:** GPT‑4o, `temperature: 0.1`.

**Exception Tool Output Parser**
- **Type / role:** Validates exception tool output.
- **Required:** `exceptionsIdentified`, `escalationRequired`, `mitigationStrategies`.

---

### 1.7 Auditability & reporting
**Overview:** Generates execution metadata, logs results to Google Sheets, consolidates outputs and sends an HTML email report to the compliance team.

**Nodes involved:**
- Consolidate Results
- Generate Audit Metadata
- Log to Audit Trail Sheet
- Send Compliance Report

**Node details**

**Generate Audit Metadata**
- **Type / role:** `code` – builds audit metadata (IDs, timestamps, lineage, checksums).
- **Configuration choices:**
  - Uses `$workflow.id`, `$execution.id`, `$execution.mode`, `$workflow.name`, `$workflow.active`.
  - Reads risk assessment via: `$('Risk Signal Agent').first().json` (important dependency).
  - Builds `dataLineage` array of processing steps.
  - Creates lightweight checksum hashes from JSON stringification.
  - Outputs enriched objects: merges each incoming compliance item with `auditMetadata`.
- **Outputs:** To **Log to Audit Trail Sheet** and **Consolidate Results**.
- **Edge cases:**
  - If `$('Risk Signal Agent').first().json` doesn’t contain direct `riskScore/riskLevel` (because agent output is under `.output`), `riskScore`/`riskLevel` stored in audit metadata may be null. (This is likely in the current workflow.)
  - `$version` may be undefined; code falls back to `'unknown'`.

**Log to Audit Trail Sheet**
- **Type / role:** `googleSheets` – persists audit trail.
- **Configuration:**
  - Operation: `appendOrUpdate`
  - Document ID: from config `auditSheetId`
  - Sheet name: “Audit Trail”
  - Matching column: `workflowExecutionId`
  - Mapping: auto-map input fields.
- **Failure types:** OAuth expired, sheet permissions, missing sheet name, mismatched columns (appendOrUpdate requires stable key column).

**Consolidate Results**
- **Type / role:** `merge` – combines compliance output with audit-enriched stream for downstream emailing.
- **Configuration:** `combineByPosition`.
- **Inputs:**
  - From **Compliance Orchestration Agent**
  - From **Generate Audit Metadata**
  - Also receives Medium risk path from **Route by Risk Level** (but note: Medium route goes to Consolidate Results index 1 without compliance agent—this can create mismatches).
- **Edge cases:** The Medium-risk branch wiring is inconsistent: it bypasses compliance agent but merges into results, likely producing missing fields expected by email.

**Send Compliance Report**
- **Type / role:** `emailSend` – sends formatted HTML report.
- **Configuration:**
  - To: config `complianceEmail`
  - From: placeholder sender email
  - Subject: `Risk Assessment & Compliance Report - YYYY-MM-DD`
  - HTML template references many fields under `$json.output.*` (risk + compliance details).
- **Edge cases:**
  - If merged data shape is not exactly as expected (`$json.output` missing), expressions like `$json.output.requiredApprovals.map(...)` will throw.
  - SMTP/Email credentials are not shown in JSON; node will require a configured email transport in n8n.
  - Some exceptionHandling fields are accessed without full null-guards.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | Schedule Trigger | Scheduled execution entry point | — | Workflow Configuration | ## Multi-Source Risk Aggregation<br>**What:** Fetches and merges financial metrics and claims data on scheduled intervals for unified risk analysis<br>**Why:** Ensures comprehensive risk coverage by correlating financial anomalies with operational claims data for complete visibility |
| Workflow Configuration | Set | Central parameter store (URLs, thresholds, recipients) | Schedule Trigger | Fetch Risk Data; Fetch Claims Data | ## Setup Steps<br>1. Configure Schedule Trigger with risk monitoring frequency<br>2. Connect Workflow Configuration node with data source parameters<br>3. Set up Fetch B2B Data and Fetch Claims Data nodes with respective API credentials<br>4. Configure Merge Financial Data node for data consolidation<br>5. Connect Calculate Risk Metrics node with risk scoring algorithms<br>6. Set up Risk Signal Agent with OpenAI/Nvidia API credentials for anomaly detection<br>7. Configure parallel output parsers<br>8. Connect Check Critical Risk node with severity routing logic<br>9. Set up Route by Risk Level node for workflow branching |
| Fetch Risk Data | HTTP Request | Pull risk/exposure dataset from API | Workflow Configuration | Merge Financial Data | ## Multi-Source Risk Aggregation<br>**What:** Fetches and merges financial metrics and claims data on scheduled intervals for unified risk analysis<br>**Why:** Ensures comprehensive risk coverage by correlating financial anomalies with operational claims data for complete visibility |
| Fetch Claims Data | HTTP Request | Pull claims dataset from API | Workflow Configuration | Merge Financial Data | ## Multi-Source Risk Aggregation<br>**What:** Fetches and merges financial metrics and claims data on scheduled intervals for unified risk analysis<br>**Why:** Ensures comprehensive risk coverage by correlating financial anomalies with operational claims data for complete visibility |
| Merge Financial Data | Merge | Combine risk + claims streams | Fetch Risk Data; Fetch Claims Data | Risk Signal Agent; Calculate Risk Metrics | ## Multi-Source Risk Aggregation<br>**What:** Fetches and merges financial metrics and claims data on scheduled intervals for unified risk analysis<br>**Why:** Ensures comprehensive risk coverage by correlating financial anomalies with operational claims data for complete visibility |
| Calculate Risk Metrics | Code | Compute concentration/volatility/trend metrics | Merge Financial Data | Risk Signal Agent | ## Multi-Source Risk Aggregation<br>**What:** Fetches and merges financial metrics and claims data on scheduled intervals for unified risk analysis<br>**Why:** Ensures comprehensive risk coverage by correlating financial anomalies with operational claims data for complete visibility |
| Risk Signal Agent | LangChain Agent | GPT-based risk scoring + factors (structured) | Merge Financial Data; Calculate Risk Metrics | Route by Risk Level; Check Critical Risk | ## AI-Powered Risk Detection<br>**What:** Processes merged data through risk signal agent with parallel output parsing for regulatory and operational risk identification<br>**Why:** Leverages AI to detect complex risk patterns and compliance violations that rule-based systems miss |
| OpenAI Model - Risk Agent | OpenAI Chat Model | LLM backend for risk agent (gpt-4o) | — (AI connection) | Risk Signal Agent | ## AI-Powered Risk Detection<br>**What:** Processes merged data through risk signal agent with parallel output parsing for regulatory and operational risk identification<br>**Why:** Leverages AI to detect complex risk patterns and compliance violations that rule-based systems miss |
| Risk Assessment Output Parser | Structured Output Parser | Enforce JSON schema for risk output | — (AI connection) | Risk Signal Agent | ## AI-Powered Risk Detection<br>**What:** Processes merged data through risk signal agent with parallel output parsing for regulatory and operational risk identification<br>**Why:** Leverages AI to detect complex risk patterns and compliance violations that rule-based systems miss |
| Check Critical Risk | IF | Gate Slack escalation for high/critical cases | Risk Signal Agent | Send Critical Alert to Slack; Route by Risk Level | ## Severity-Based Orchestration<br>**What:** Routes risks through severity-specific workflows with parallel processing for compliance documentation, stakeholder alerts, and audit logging<br>**Why:** Ensures critical risks receive immediate multi-channel escalation while standard risks follow proper documentation procedures without bottlenecks |
| Send Critical Alert to Slack | Slack | Immediate critical alerting | Check Critical Risk | Merge Critical Path | ## Severity-Based Orchestration<br>**What:** Routes risks through severity-specific workflows with parallel processing for compliance documentation, stakeholder alerts, and audit logging<br>**Why:** Ensures critical risks receive immediate multi-channel escalation while standard risks follow proper documentation procedures without bottlenecks |
| Route by Risk Level | Switch | Branch by score (High/Medium/Low) | Risk Signal Agent; Check Critical Risk | Compliance Orchestration Agent; Merge Critical Path; Consolidate Results | ## Severity-Based Orchestration<br>**What:** Routes risks through severity-specific workflows with parallel processing for compliance documentation, stakeholder alerts, and audit logging<br>**Why:** Ensures critical risks receive immediate multi-channel escalation while standard risks follow proper documentation procedures without bottlenecks |
| Merge Critical Path | Merge | Recombine post-Slack + routing into one stream | Send Critical Alert to Slack; Route by Risk Level | Compliance Orchestration Agent | ## Severity-Based Orchestration<br>**What:** Routes risks through severity-specific workflows with parallel processing for compliance documentation, stakeholder alerts, and audit logging<br>**Why:** Ensures critical risks receive immediate multi-channel escalation while standard risks follow proper documentation procedures without bottlenecks |
| Compliance Orchestration Agent | LangChain Agent | Coordinate compliance actions, approvals, audit package | Merge Critical Path; Route by Risk Level | Consolidate Results; Generate Audit Metadata | ## Severity-Based Orchestration<br>**What:** Routes risks through severity-specific workflows with parallel processing for compliance documentation, stakeholder alerts, and audit logging<br>**Why:** Ensures critical risks receive immediate multi-channel escalation while standard risks follow proper documentation procedures without bottlenecks |
| OpenAI Model - Compliance Agent | OpenAI Chat Model | LLM backend for compliance agent (gpt-4o) | — (AI connection) | Compliance Orchestration Agent | ## Severity-Based Orchestration<br>**What:** Routes risks through severity-specific workflows with parallel processing for compliance documentation, stakeholder alerts, and audit logging<br>**Why:** Ensures critical risks receive immediate multi-channel escalation while standard risks follow proper documentation procedures without bottlenecks |
| Compliance Output Parser | Structured Output Parser | Enforce JSON schema for compliance output | — (AI connection) | Compliance Orchestration Agent | ## Severity-Based Orchestration<br>**What:** Routes risks through severity-specific workflows with parallel processing for compliance documentation, stakeholder alerts, and audit logging<br>**Why:** Ensures critical risks receive immediate multi-channel escalation while standard risks follow proper documentation procedures without bottlenecks |
| Regulatory Analysis Agent Tool | Agent Tool | Tool: regulatory framework analysis | — (AI tool connection) | Compliance Orchestration Agent | ## Severity-Based Orchestration<br>**What:** Routes risks through severity-specific workflows with parallel processing for compliance documentation, stakeholder alerts, and audit logging<br>**Why:** Ensures critical risks receive immediate multi-channel escalation while standard risks follow proper documentation procedures without bottlenecks |
| OpenAI Model - Regulatory Tool | OpenAI Chat Model | LLM backend for regulatory tool | — (AI connection) | Regulatory Analysis Agent Tool | ## Severity-Based Orchestration<br>**What:** Routes risks through severity-specific workflows with parallel processing for compliance documentation, stakeholder alerts, and audit logging<br>**Why:** Ensures critical risks receive immediate multi-channel escalation while standard risks follow proper documentation procedures without bottlenecks |
| Regulatory Tool Output Parser | Structured Output Parser | Enforce JSON schema for regulatory tool output | — (AI connection) | Regulatory Analysis Agent Tool | ## Severity-Based Orchestration<br>**What:** Routes risks through severity-specific workflows with parallel processing for compliance documentation, stakeholder alerts, and audit logging<br>**Why:** Ensures critical risks receive immediate multi-channel escalation while standard risks follow proper documentation procedures without bottlenecks |
| Exception Handler Agent Tool | Agent Tool | Tool: exception identification/escalation | — (AI tool connection) | Compliance Orchestration Agent | ## Severity-Based Orchestration<br>**What:** Routes risks through severity-specific workflows with parallel processing for compliance documentation, stakeholder alerts, and audit logging<br>**Why:** Ensures critical risks receive immediate multi-channel escalation while standard risks follow proper documentation procedures without bottlenecks |
| OpenAI Model - Exception Tool | OpenAI Chat Model | LLM backend for exception tool | — (AI connection) | Exception Handler Agent Tool | ## Severity-Based Orchestration<br>**What:** Routes risks through severity-specific workflows with parallel processing for compliance documentation, stakeholder alerts, and audit logging<br>**Why:** Ensures critical risks receive immediate multi-channel escalation while standard risks follow proper documentation procedures without bottlenecks |
| Exception Tool Output Parser | Structured Output Parser | Enforce JSON schema for exception tool output | — (AI connection) | Exception Handler Agent Tool | ## Severity-Based Orchestration<br>**What:** Routes risks through severity-specific workflows with parallel processing for compliance documentation, stakeholder alerts, and audit logging<br>**Why:** Ensures critical risks receive immediate multi-channel escalation while standard risks follow proper documentation procedures without bottlenecks |
| Generate Audit Metadata | Code | Build audit metadata + checksums + lineage | Compliance Orchestration Agent | Log to Audit Trail Sheet; Consolidate Results | ## Severity-Based Orchestration<br>**What:** Routes risks through severity-specific workflows with parallel processing for compliance documentation, stakeholder alerts, and audit logging<br>**Why:** Ensures critical risks receive immediate multi-channel escalation while standard risks follow proper documentation procedures without bottlenecks |
| Log to Audit Trail Sheet | Google Sheets | Persist audit trail (append/update) | Generate Audit Metadata | — | ## Severity-Based Orchestration<br>**What:** Routes risks through severity-specific workflows with parallel processing for compliance documentation, stakeholder alerts, and audit logging<br>**Why:** Ensures critical risks receive immediate multi-channel escalation while standard risks follow proper documentation procedures without bottlenecks |
| Consolidate Results | Merge | Combine compliance + audit outputs for reporting | Compliance Orchestration Agent; Generate Audit Metadata; Route by Risk Level | Send Compliance Report | ## Severity-Based Orchestration<br>**What:** Routes risks through severity-specific workflows with parallel processing for compliance documentation, stakeholder alerts, and audit logging<br>**Why:** Ensures critical risks receive immediate multi-channel escalation while standard risks follow proper documentation procedures without bottlenecks |
| Send Compliance Report | Email Send | Email the HTML compliance/risk report | Consolidate Results | — | ## Severity-Based Orchestration<br>**What:** Routes risks through severity-specific workflows with parallel processing for compliance documentation, stakeholder alerts, and audit logging<br>**Why:** Ensures critical risks receive immediate multi-channel escalation while standard risks follow proper documentation procedures without bottlenecks |
| Sticky Note | Sticky Note | Commentary | — | — | ## Prerequisites<br>OpenAI or Nvidia API credentials for AI-powered risk analysis, financial data API access<br>## Use Cases<br>Insurance companies monitoring claims fraud patterns, financial institutions detecting transaction anomalies<br>## Customization<br>Adjust risk scoring algorithms for industry-specific thresholds<br>## Benefits<br>Reduces risk detection time by 80%, eliminates manual compliance monitoring |
| Sticky Note1 | Sticky Note | Commentary | — | — | ## Setup Steps<br>1. Configure Schedule Trigger with risk monitoring frequency<br>2. Connect Workflow Configuration node with data source parameters<br>3. Set up Fetch B2B Data and Fetch Claims Data nodes with respective API credentials<br>4. Configure Merge Financial Data node for data consolidation<br>5. Connect Calculate Risk Metrics node with risk scoring algorithms<br>6. Set up Risk Signal Agent with OpenAI/Nvidia API credentials for anomaly detection<br>7. Configure parallel output parsers<br>8. Connect Check Critical Risk node with severity routing logic<br>9. Set up Route by Risk Level node for workflow branching |
| Sticky Note2 | Sticky Note | Commentary | — | — | ## How It Works<br>This workflow automates comprehensive risk signal detection and regulatory compliance management across financial and claims data sources. Designed for risk management teams, compliance officers, and financial auditors, it solves the critical challenge of identifying potential risks while ensuring timely regulatory reporting and stakeholder notifications.<br>The system operates on scheduled intervals, fetching data from multiple sources including financial APIs and claims databases, then merging these streams for unified analysis. It employs an AI-powered risk signal agent to detect anomalies, regulatory violations, and compliance issues. The workflow intelligently routes findings based on risk severity, orchestrating parallel processes for critical risks requiring immediate escalation and standard risks needing documentation. It manages multi-channel notifications through Slack and email, generates comprehensive compliance documentation, and maintains detailed audit trails. By coordinating regulatory analysis, exception handling, and evidence collection, it ensures complete risk visibility while automating compliance workflows. |
| Sticky Note3 | Sticky Note | Commentary | — | — | ## AI-Powered Risk Detection<br>**What:** Processes merged data through risk signal agent with parallel output parsing for regulatory and operational risk identification<br>**Why:** Leverages AI to detect complex risk patterns and compliance violations that rule-based systems miss |
| Sticky Note4 | Sticky Note | Commentary | — | — | ## Severity-Based Orchestration<br>**What:** Routes risks through severity-specific workflows with parallel processing for compliance documentation, stakeholder alerts, and audit logging<br>**Why:** Ensures critical risks receive immediate multi-channel escalation while standard risks follow proper documentation procedures without bottlenecks |
| Sticky Note5 | Sticky Note | Commentary | — | — | ## Multi-Source Risk Aggregation<br>**What:** Fetches and merges financial metrics and claims data on scheduled intervals for unified risk analysis<br>**Why:** Ensures comprehensive risk coverage by correlating financial anomalies with operational claims data for complete visibility |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** named **“Smart Risk Assessment and Compliance Orchestration Automation System”** (or your preferred name). Keep it inactive until credentials are set.

2) **Add “Schedule Trigger”**
   - Type: *Schedule Trigger*
   - Set it to run **daily at 09:00** (or your desired cadence/timezone).

3) **Add “Workflow Configuration”**
   - Type: *Set*
   - Add fields:
     - `riskDataApiUrl` (string) – risk/exposure endpoint
     - `claimsDataApiUrl` (string) – claims endpoint
     - `complianceEmail` (string)
     - `riskThresholdHigh` (number) = 75
     - `riskThresholdMedium` (number) = 50
     - `slackChannel` (string) – Slack channel ID
     - `auditSheetId` (string) – Google Sheets document ID
     - `criticalRiskThreshold` (number) = 90 (optional unless you update routing)
   - Enable **Include Other Fields**.
   - Connect: **Schedule Trigger → Workflow Configuration**.

4) **Add HTTP nodes**
   - **Fetch Risk Data**
     - Type: *HTTP Request*
     - URL: expression from configuration (`riskDataApiUrl`)
     - Header: `Content-Type: application/json`
   - **Fetch Claims Data**
     - Same, using `claimsDataApiUrl`
   - Connect: **Workflow Configuration → Fetch Risk Data** and **Workflow Configuration → Fetch Claims Data**.

5) **Add “Merge Financial Data”**
   - Type: *Merge*
   - Mode: **Combine**
   - Combine By: **Position**
   - Connect:
     - **Fetch Risk Data → Merge Financial Data (Input 1 / index 0)**
     - **Fetch Claims Data → Merge Financial Data (Input 2 / index 1)**

6) **Add “Calculate Risk Metrics”**
   - Type: *Code*
   - Paste the JS logic that aggregates exposure/claims, HHI indices, volatility, trend, and returns a single item.
   - Connect: **Merge Financial Data → Calculate Risk Metrics**.

7) **Add “Risk Signal Agent” + model + parser**
   - Node: *AI Agent* (LangChain Agent)
     - Prompt text: `Risk Data: {{ JSON.stringify($json) }}`
     - System message: risk scoring instructions + constraints
     - Enable structured output parsing.
   - Add **OpenAI Chat Model** node:
     - Model: **gpt-4o**
     - Temperature: **0.2**
     - Credentials: create/select **OpenAI API** credential.
     - Connect to agent via **ai_languageModel**.
   - Add **Structured Output Parser**:
     - Provide the manual JSON Schema for risk output (riskScore, riskLevel, keyRiskFactors, reasoning, etc.).
     - Connect via **ai_outputParser**.
   - Connect inputs:
     - **Merge Financial Data → Risk Signal Agent**
     - **Calculate Risk Metrics → Risk Signal Agent**
     (This matches the original design, though you may prefer a single consolidated payload to avoid duplicate runs.)

8) **Add “Check Critical Risk”**
   - Type: *IF*
   - Condition: `{{$json.riskScore}} >= {{$('Workflow Configuration').first().json.riskThresholdHigh}}`
   - Connect: **Risk Signal Agent → Check Critical Risk**.

9) **Add “Send Critical Alert to Slack”**
   - Type: *Slack*
   - Auth: **OAuth2**
   - Credentials: create/select Slack OAuth2 credential with permission to post in the target channel.
   - Channel: expression from `slackChannel`
   - Text: compose message using `$json.output.riskScore`, `$json.output.keyRiskFactors`, etc. (as in the workflow).
   - Connect: **Check Critical Risk (true) → Slack**.

10) **Add “Route by Risk Level”**
   - Type: *Switch*
   - Add outputs:
     - High Risk: `riskScore >= 75`
     - Medium Risk: `50 <= riskScore < 75`
     - Fallback renamed: Low Risk
   - Connect:  
     - **Risk Signal Agent → Route by Risk Level**  
     - **Check Critical Risk (false) → Route by Risk Level**

11) **Add “Merge Critical Path”**
   - Type: *Merge* (Combine by position)
   - Connect:
     - **Slack → Merge Critical Path**
     - **Route by Risk Level (High Risk output) → Merge Critical Path**
   - Output goes to the compliance agent (next step).

12) **Add Compliance orchestration agent + tools**
   - **Compliance Orchestration Agent** (AI Agent)
     - Text: `Risk Assessment: {{ JSON.stringify($json.output) }}`
     - System message: compliance coordination + audit packaging constraints
     - Tools: add two *Agent Tool* nodes and connect them via **ai_tool**
     - Add OpenAI model node (gpt-4o, temperature 0.2) via **ai_languageModel**
     - Add Structured Output Parser with schema requiring workflowActions, requiredApprovals, regulatoryItems, auditPackage, summary via **ai_outputParser**
   - **Regulatory Analysis Agent Tool**
     - Add OpenAI model (gpt-4o, temperature 0.1)
     - Add output parser schema requiring applicableFrameworks + complianceRequirements
   - **Exception Handler Agent Tool**
     - Add OpenAI model (gpt-4o, temperature 0.1)
     - Add output parser schema requiring exceptionsIdentified + escalationRequired + mitigationStrategies
   - Connect compliance agent inputs:
     - **Merge Critical Path → Compliance Orchestration Agent**
     - **Route by Risk Level (High Risk output) → Compliance Orchestration Agent** (as in original)

13) **Add “Generate Audit Metadata”**
   - Type: *Code*
   - Use code to:
     - Read `$execution`/`$workflow` context
     - Build lineage + checksums
     - Attach `auditMetadata` to the compliance agent output
   - Connect: **Compliance Orchestration Agent → Generate Audit Metadata**.

14) **Add “Log to Audit Trail Sheet”**
   - Type: *Google Sheets*
   - Credentials: Google Sheets OAuth2
   - Operation: **Append or Update**
   - Document ID: from `auditSheetId`
   - Sheet name: “Audit Trail”
   - Matching column: `workflowExecutionId`
   - Connect: **Generate Audit Metadata → Google Sheets**

15) **Add “Consolidate Results”**
   - Type: *Merge* (Combine by position)
   - Connect:
     - **Compliance Orchestration Agent → Consolidate Results**
     - **Generate Audit Metadata → Consolidate Results**

16) **Add “Send Compliance Report”**
   - Type: *Email Send*
   - To: expression from `complianceEmail`
   - From: your sender address
   - Subject: `Risk Assessment & Compliance Report - {{ $now.toFormat('yyyy-MM-dd') }}`
   - HTML: use the provided template referencing `$json.output.*`
   - Connect: **Consolidate Results → Send Compliance Report**
   - Configure email transport/credentials depending on your n8n setup (SMTP or provider integration).

17) **(Recommended fixes while reproducing)**
   - Use `criticalRiskThreshold` (90) in **Check Critical Risk** if you truly want “critical” behavior.
   - Ensure Low/Medium risk paths still generate audit logs and reports (currently inconsistent).
   - Update **Generate Audit Metadata** to read `$('Risk Signal Agent').first().json.output` if that’s where `riskScore` actually lives.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| **Prerequisites:** OpenAI or Nvidia API credentials for AI-powered risk analysis, financial data API access | From sticky note “Prerequisites” |
| **Use cases:** Insurance companies monitoring claims fraud patterns; financial institutions detecting transaction anomalies | From sticky note “Use Cases” |
| **Customization:** Adjust risk scoring algorithms for industry-specific thresholds | From sticky note “Customization” |
| **Benefits:** Reduces risk detection time by 80%, eliminates manual compliance monitoring | From sticky note “Benefits” |
| “How It Works” high-level description of scheduled multi-source fetch → AI risk detection → severity routing → Slack/email → audit trails | From sticky note “How It Works” |


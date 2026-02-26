Detect transaction fraud and manage compliance with GPT-4 and Airtable

https://n8nworkflows.xyz/workflows/detect-transaction-fraud-and-manage-compliance-with-gpt-4-and-airtable-13598


# Detect transaction fraud and manage compliance with GPT-4 and Airtable

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title (given):** Detect transaction fraud and manage compliance with GPT-4 and Airtable  
**Workflow name (JSON):** AI-powered transaction fraud detection and compliance monitoring

**Purpose:**  
Runs on a schedule to pull “Pending” blockchain transaction records from Airtable, uses GPT-4o via n8n’s LangChain nodes to validate/extract transaction signals, classifies risk, orchestrates compliance actions (including deep investigation, refined risk scoring, and report generation through specialist agent tools), merges results, then updates the transaction record back in Airtable with a final action, score, and report references.

**Target use cases:**  
AML/KYC monitoring, suspicious activity detection, automated case triage, compliance reporting pipelines for fintech/crypto operations handling high transaction volumes.

### Logical Blocks
1.1 **Scheduling & Global Parameters**  
Runs every 15 minutes and sets workflow-wide thresholds/framework parameters.

1.2 **Transaction Intake (Airtable Queue)**  
Pulls Airtable transaction rows where `Status = "Pending"`.

1.3 **AI Transaction Signal Validation & Initial Risk Classification**  
GPT extracts key signals, detects anomalies, assigns `riskScore` and `riskLevel`, then routes by risk level.

1.4 **Compliance Orchestration (Agent with Tools)**  
A Compliance Agent coordinates tool-calls to Investigation/Risk Scoring/Reporting and an Airtable compliance-records tool, returning a structured compliance decision.

1.5 **Consolidation & Persistence**  
Merges agent outputs and updates the Airtable transaction row with final status/action, risk score, timestamps, and summaries.

---

## 2. Block-by-Block Analysis

### 2.1 Scheduling & Global Parameters

**Overview:**  
Triggers workflow execution every 15 minutes and sets configuration variables (risk thresholds, compliance framework, investigation depth) that can be referenced downstream.

**Nodes involved:**  
- Schedule Trigger  
- Workflow Configuration

#### Node: Schedule Trigger
- **Type / role:** `n8n-nodes-base.scheduleTrigger` — time-based entry point.
- **Configuration:** Runs every **15 minutes**.
- **Inputs/Outputs:** No input; outputs to **Workflow Configuration**.
- **Version notes:** typeVersion **1.3**.
- **Failure/edge cases:** Minimal; if n8n instance is paused/offline, schedules won’t run; drift can occur under heavy load.

#### Node: Workflow Configuration
- **Type / role:** `n8n-nodes-base.set` — injects workflow parameters.
- **Configuration choices:**
  - Adds/overwrites fields while **including other fields** (`includeOtherFields: true`).
  - Sets:
    - `riskThresholdHigh` = **80**
    - `riskThresholdMedium` = **50**
    - `complianceFramework` = **"AML/KYC"**
    - `investigationDepth` = **"comprehensive"**
- **Key variables:** These become part of the JSON payload passed forward.
- **Inputs/Outputs:** Input from **Schedule Trigger**; output to **Fetch Pending Transactions**.
- **Version notes:** typeVersion **3.4**.
- **Failure/edge cases:** If later nodes expect these values but upstream data overwrites them, outcomes can shift; ensure field names don’t collide with Airtable row fields.

---

### 2.2 Transaction Intake (Airtable Queue)

**Overview:**  
Queries Airtable for transactions awaiting review (`Status = "Pending"`), producing a list of transaction records to be assessed.

**Nodes involved:**  
- Fetch Pending Transactions

#### Node: Fetch Pending Transactions
- **Type / role:** `n8n-nodes-base.airtable` — data retrieval from Airtable.
- **Configuration choices:**
  - **Operation:** Search
  - **Filter formula:** `={Status} = "Pending"`
  - **Base/Table:** placeholders:
    - Base ID: `<__PLACEHOLDER_VALUE__Airtable Base ID__>`
    - Transactions Table ID: `<__PLACEHOLDER_VALUE__Transactions Table ID__>`
- **Inputs/Outputs:** Input from **Workflow Configuration**; output to **Transaction Signal Agent**.
- **Version notes:** typeVersion **2.1**.
- **Failure/edge cases:**
  - Auth/permission errors (invalid token, missing base/table access).
  - Formula mismatch if field name differs (must be exactly `Status`).
  - Pagination/record limits depending on Airtable settings; large queues may require batching.

---

### 2.3 AI Transaction Signal Validation & Initial Risk Classification

**Overview:**  
Uses an LLM agent to validate transaction structure, extract key signals, detect anomalies, compute an initial risk score and risk level, then routes records into risk categories.

**Nodes involved:**  
- Transaction Signal Agent  
- OpenAI Model - Transaction Signal  
- Transaction Signal Parser  
- Route by Risk Level

#### Node: OpenAI Model - Transaction Signal
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — LLM backend for the agent.
- **Configuration choices:**
  - **Model:** `gpt-4o`
  - Uses OpenAI credential: **“OpenAi account”**
- **Inputs/Outputs:** Connected via `ai_languageModel` into **Transaction Signal Agent**.
- **Version notes:** typeVersion **1.3**.
- **Failure/edge cases:** API quota/rate limits, model availability, network timeouts, credential misconfiguration.

#### Node: Transaction Signal Parser
- **Type / role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — enforces JSON schema output from the agent.
- **Configuration choices:**
  - Manual JSON schema requiring:
    - `transactionId`, `validationStatus` (valid/invalid/incomplete)
    - `riskScore` (0–100), `riskLevel` (low/medium/high)
    - `signals` object (amount, sender, receiver, timestamp, chain, contractAddress)
    - `anomalies` array, `requiresInvestigation` boolean, `reasoning`
- **Inputs/Outputs:** Feeds parsed output into **Transaction Signal Agent** via `ai_outputParser`.
- **Version notes:** typeVersion **1.3**.
- **Failure/edge cases:** If the model returns non-conforming JSON or misses required fields, parsing fails; consider adding retries or fallback logic.

#### Node: Transaction Signal Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — performs transaction signal extraction and initial risk assessment.
- **Configuration choices:**
  - **Text input:** `={{ $json }}` (passes the full Airtable transaction record JSON to the agent)
  - **System message:** specialized “Transaction Signal Validation Agent” instructions (validate structure, extract signals, identify anomalies, assign risk).
  - **Output parser enabled:** yes (the structured parser above)
- **Inputs/Outputs:**
  - Main input from **Fetch Pending Transactions**
  - `ai_languageModel` from **OpenAI Model - Transaction Signal**
  - `ai_outputParser` from **Transaction Signal Parser**
  - Main output to **Route by Risk Level**
- **Version notes:** typeVersion **3.1**.
- **Failure/edge cases:**
  - Garbage-in risk: if Airtable rows lack required on-chain fields, `validationStatus` may be `incomplete/invalid`.
  - If `transactionId` is not present in the Airtable record, the LLM must infer it—this can produce mismatches later when updating records.

#### Node: Route by Risk Level
- **Type / role:** `n8n-nodes-base.switch` — branches by risk category.
- **Configuration choices:**
  - Rules:
    - If `{{$json.riskLevel}} == "high"` → output “High Risk”
    - If `... == "medium"` → “Medium Risk”
    - If `... == "low"` → “Low Risk”
  - Fallback output renamed to **“Unclassified”**
- **Inputs/Outputs:** Input from **Transaction Signal Agent**; all three risk outputs route to **Compliance Agent** (same target).
- **Version notes:** typeVersion **3.4**.
- **Failure/edge cases:** If `riskLevel` is missing/typo, item goes to “Unclassified” output; currently that fallback is **not connected** in this workflow (so such items would be dropped unless connected).

---

### 2.4 Compliance Orchestration (Agent with Tools)

**Overview:**  
A central Compliance Agent uses GPT-4o and can call specialist agent tools (Investigation, Risk Scoring, Reporting) plus an Airtable compliance-record tool. It returns a structured compliance decision (approve/flag/escalate/block) with supporting details.

**Nodes involved:**  
- Compliance Agent  
- OpenAI Model - Compliance  
- Compliance Action Parser  
- Investigation Agent Tool + OpenAI Model - Investigation + Investigation Parser  
- Risk Scoring Agent Tool + OpenAI Model - Risk Scoring + Risk Score Parser  
- Reporting Agent Tool + OpenAI Model - Reporting + Report Parser  
- Airtable Tool - Compliance Records

#### Node: OpenAI Model - Compliance
- **Type / role:** LLM backend for compliance orchestration (`lmChatOpenAi`).
- **Configuration:** `gpt-4o` using the same OpenAI credential.
- **Connections:** `ai_languageModel` → **Compliance Agent**
- **Failure/edge cases:** rate limits/timeouts; tool-calling can increase latency.

#### Node: Compliance Action Parser
- **Type / role:** Structured parser for the Compliance Agent output.
- **Schema requires:**
  - `transactionId`
  - `complianceAction`: approve/flag/escalate/block
  - `finalRiskScore` (0–100)
  - `investigationSummary`
  - `complianceFrameworks` (array)
  - `requiredActions` (array)
  - `escalationRequired` boolean
  - `reportGenerated` boolean
  - `reasoning`
- **Connections:** `ai_outputParser` → **Compliance Agent**
- **Edge cases:** parsing failures if the agent omits required fields or uses invalid enum values.

#### Node: Airtable Tool - Compliance Records
- **Type / role:** `n8n-nodes-base.airtableTool` — exposes Airtable search/update capability as an agent tool.
- **Configuration:**
  - **Operation:** Search
  - Base/Table placeholders:
    - Base ID: `<__PLACEHOLDER_VALUE__Airtable Base ID__>`
    - Compliance Records Table ID: `<__PLACEHOLDER_VALUE__Compliance Records Table ID__>`
  - Tool description indicates both query/update, but the configured operation is **search**.
- **Connections:** `ai_tool` → **Compliance Agent**
- **Edge cases:**
  - If the Compliance Agent expects to *update* records but tool is configured only for *search*, tool usefulness is limited.
  - Airtable permissions/table schema mismatches.

#### Node: Investigation Agent Tool
- **Type / role:** `@n8n/n8n-nodes-langchain.agentTool` — a callable specialist tool the Compliance Agent can invoke.
- **Configuration:**
  - **Tool input text:** `{{ $fromAI("transactionData", "Transaction data and signals for investigation", "json") }}`
    - This means the Compliance Agent must provide a `transactionData` argument when calling the tool.
  - **System message:** deep investigation guidance (threat DB checks, fund flow, layering patterns, sanctions involvement).
  - **Has output parser:** yes.
- **Connections:**
  - `ai_languageModel` from **OpenAI Model - Investigation**
  - `ai_outputParser` from **Investigation Parser**
  - `ai_tool` into **Compliance Agent**
- **Edge cases:**
  - If the Compliance Agent doesn’t pass `transactionData`, tool call will fail.
  - Hallucinated “threat database” checks unless you later replace/augment with real enrichment APIs.

#### Node: OpenAI Model - Investigation
- **Type / role:** LLM backend (`gpt-4o`) for Investigation tool.
- **Connections:** `ai_languageModel` → **Investigation Agent Tool**
- **Edge cases:** same OpenAI API concerns; investigation prompts are long → token usage.

#### Node: Investigation Parser
- **Type / role:** Structured parser for investigation results.
- **Schema requires:**
  - `investigationId`, `findings[]`
  - `threatLevel`: none/low/medium/high/critical
  - `evidenceQuality`: weak/moderate/strong/conclusive
  - `suspiciousPatterns[]`, `addressFlags[]`, `recommendations[]`, `reasoning`
- **Connections:** `ai_outputParser` → **Investigation Agent Tool**
- **Edge cases:** strict enum validation can fail if model returns e.g. “very high”.

#### Node: Risk Scoring Agent Tool
- **Type / role:** Specialist tool for refined scoring based on transaction + investigation context.
- **Configuration:**
  - Input mapping: `{{ $fromAI("transactionData", "Transaction data and investigation findings for risk scoring", "json") }}`
  - System message: multi-factor scoring model + confidence + mitigation.
- **Connections:**
  - `ai_languageModel` from **OpenAI Model - Risk Scoring**
  - `ai_outputParser` from **Risk Score Parser**
  - `ai_tool` into **Compliance Agent**
- **Edge cases:** requires the Compliance Agent to pass the correct JSON bundle; parser strictness.

#### Node: OpenAI Model - Risk Scoring
- **Type / role:** LLM backend (`gpt-4o`) for risk scoring tool.

#### Node: Risk Score Parser
- **Type / role:** Structured parser for refined risk scoring.
- **Schema requires:**
  - `refinedRiskScore` (0–100), `confidenceLevel` (low/medium/high)
  - `primaryRiskDrivers[]`
  - `riskFactors` object with numeric sub-scores: amountRisk/velocityRisk/counterpartyRisk/geographyRisk
  - `mitigationStrategies[]`, `reasoning`
- **Edge cases:** numeric fields often come back as strings if the model drifts; strict parser may fail.

#### Node: Reporting Agent Tool
- **Type / role:** Specialist tool to generate compliance/SAR-style reports.
- **Configuration:**
  - Input mapping: `{{ $fromAI("complianceData", "Compliance assessment and investigation data for reporting", "json") }}`
  - System message: AML/KYC, SAR format, audit trail metadata.
- **Connections:**
  - `ai_languageModel` from **OpenAI Model - Reporting**
  - `ai_outputParser` from **Report Parser**
  - `ai_tool` into **Compliance Agent**
- **Edge cases:** report length may be large; cost/token issues; if the Compliance Agent never calls this tool, `reportId` might not exist downstream unless Compliance Agent fabricates it.

#### Node: OpenAI Model - Reporting
- **Type / role:** LLM backend (`gpt-4o`) for reporting tool.

#### Node: Report Parser
- **Type / role:** Structured parser for reporting outputs.
- **Schema requires:**
  - `reportId`, `reportType` (investigation/compliance/SAR/summary)
  - `executiveSummary`, `findings[]`, `evidence[]`, `recommendations[]`, `nextSteps[]`
  - `severity` (low/medium/high/critical), `timestamp`, `reasoning`
- **Edge cases:** timestamp format ambiguity; enum strictness.

#### Node: Compliance Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — orchestrates compliance decisioning and tool use.
- **Configuration:**
  - **Text input:** `={{ $json }}` (takes the routed transaction signal object)
  - **System message:** instructs agent to:
    - assess compliance requirements (AML/KYC),
    - call investigation/risk/reporting tools,
    - query/update Airtable records,
    - decide final action (approve/flag/escalate/block),
    - return structured output.
  - **Output parser enabled:** yes (Compliance Action Parser).
- **Connections:**
  - Main input from **Route by Risk Level** (all three risk branches)
  - `ai_languageModel` from **OpenAI Model - Compliance**
  - `ai_tool` from:
    - Investigation Agent Tool
    - Risk Scoring Agent Tool
    - Reporting Agent Tool
    - Airtable Tool - Compliance Records
  - Main output goes to **Merge Results** (but see note below).
- **Important structural note (merge behavior):**
  - The workflow connects **Compliance Agent → Merge Results** to **three different merge inputs (0,1,2)** simultaneously. This duplicates the same output into all merge inputs unless n8n’s execution semantics differ in your version. Typically, Merge with `numberInputs: 3` expects *three distinct streams*, not three copies of one.
- **Version notes:** typeVersion **3.1**.
- **Failure/edge cases:**
  - Tool invocation failures (missing args for `$fromAI` variables).
  - Agent may not call tools consistently; outputs may be under-informed but still parseable.
  - High latency: tool-calling can chain multiple LLM calls.

---

### 2.5 Consolidation & Persistence

**Overview:**  
Combines outputs, then updates the originating Airtable transaction record with compliance action, final risk score, report ID, and audit fields.

**Nodes involved:**  
- Merge Results  
- Update Transaction Records

#### Node: Merge Results
- **Type / role:** `n8n-nodes-base.merge` — combines multiple input streams.
- **Configuration:** `numberInputs: 3`.
- **Connections:** Inputs 0/1/2 all come from **Compliance Agent**; output goes to **Update Transaction Records**.
- **Version notes:** typeVersion **3.2**.
- **Failure/edge cases:**
  - If Merge is configured to wait for all inputs, it may block or behave unexpectedly when all three inputs are the same stream.
  - Potential duplication of updates (same record updated multiple times).

#### Node: Update Transaction Records
- **Type / role:** `n8n-nodes-base.airtable` — writes results back to Airtable.
- **Configuration:**
  - **Operation:** Update
  - **Target:** Transactions base/table placeholders (same as fetch node).
  - **Field mappings:**
    - `Status` = `{{$json.complianceAction}}`
    - `Report ID` = `{{$json.reportId}}`
    - `Risk Score` = `{{$json.finalRiskScore}}`
    - `Updated At` = `{{$now.toISO()}}`
    - `Compliance Action` = `{{$json.complianceAction}}`
    - `Investigation Summary` = `{{$json.investigationSummary}}`
- **Inputs/Outputs:** Input from **Merge Results**; end of workflow.
- **Version notes:** typeVersion **2.1**.
- **Failure/edge cases (critical):**
  - Airtable **Update** requires a record identifier (Record ID) or a configured mapping. In this node configuration, no explicit record ID field is shown; if not provided implicitly by prior Airtable node output, updates may fail or update the wrong record.
  - `reportId` is referenced, but **Compliance Agent schema does not include `reportId`** (it includes `reportGenerated` only). Unless the Merge payload includes `reportId` from the Reporting tool output, this expression may be undefined.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | scheduleTrigger | Scheduled workflow entry | — | Workflow Configuration | ## How It Works  This workflow automates financial transaction monitoring, fraud detection, and regulatory compliance using OpenAI GPT-4 across coordinated specialist agents. It targets compliance officers, fraud analysts, and fintech operations teams managing high transaction volumes where manual review is too slow to catch emerging fraud patterns and compliance breaches in time. On schedule, the system fetches pending transactions from Airtable and routes them through a Transaction Signal Agent that classifies each by risk level—High, Medium, Low, or Unclassified. A Compliance Agent then coordinates three specialist agents: Investigation, Risk Scoring, and Reporting. Airtable stores all compliance records throughout. Results merge and update transaction records directly, giving compliance teams a fully automated, audit-ready pipeline that flags fraud, scores risk, and generates regulatory reports without manual intervention. |
| Workflow Configuration | set | Define thresholds/framework parameters | Schedule Trigger | Fetch Pending Transactions | ## Setup Steps  1. Import workflow JSON into your n8n instance. 2. Add OpenAI API credentials. 3. Set Schedule Trigger frequency aligned to your transaction processing cycle. 4. Update Workflow Configuration node with risk thresholds and compliance rule parameters. 5. Connect Airtable credentials and configure base/table IDs for Fetch Pending Transactions. |
| Fetch Pending Transactions | airtable | Pull pending transactions queue | Workflow Configuration | Transaction Signal Agent | ## Fetch Pending Transactions **What:** Retrieves unreviewed transaction records from Airtable. **Why:** Provides a structured, live transaction queue for AI risk analysis. |
| Transaction Signal Agent | langchain.agent | Validate tx structure, extract signals, initial risk | Fetch Pending Transactions; OpenAI Model - Transaction Signal; Transaction Signal Parser | Route by Risk Level | ## Specialist Agent Processing **What:** Investigation, Risk Scoring, and Reporting agents run per risk routing path. **Why:** Each agent targets a distinct compliance function, improving detection accuracy. |
| OpenAI Model - Transaction Signal | lmChatOpenAi | LLM for signal agent | — | Transaction Signal Agent | ## Prerequisites n8n (cloud or self-hosted), OpenAI API key (GPT-4), Airtable account with configured base and appropriate table schema  ## Use Cases Compliance teams automating AML screening and suspicious transaction flagging across high transaction volumes  ## Customization Replace OpenAI GPT-4 with Anthropic Claude or NVIDIA NIM in any agent node  ## Benefits Automates end-to-end fraud detection and compliance reporting, eliminating manual transaction reviews |
| Transaction Signal Parser | outputParserStructured | Enforce structured signal output | — | Transaction Signal Agent | ## Specialist Agent Processing **What:** Investigation, Risk Scoring, and Reporting agents run per risk routing path. **Why:** Each agent targets a distinct compliance function, improving detection accuracy. |
| Route by Risk Level | switch | Branch by low/medium/high risk | Transaction Signal Agent | Compliance Agent | ## Specialist Agent Processing **What:** Investigation, Risk Scoring, and Reporting agents run per risk routing path. **Why:** Each agent targets a distinct compliance function, improving detection accuracy. |
| Compliance Agent | langchain.agent | Orchestrate compliance decision + tool calls | Route by Risk Level; OpenAI Model - Compliance; Compliance Action Parser; Agent Tools; Airtable Tool | Merge Results (3 inputs) | ## Compliance Records Storage **What:** Airtable Tool logs all compliance findings and agent outputs throughout processing. **Why:** Maintains a real-time, audit-ready record of every transaction reviewed. |
| OpenAI Model - Compliance | lmChatOpenAi | LLM for compliance orchestration | — | Compliance Agent | ## Prerequisites n8n (cloud or self-hosted), OpenAI API key (GPT-4), Airtable account with configured base and appropriate table schema  ## Use Cases Compliance teams automating AML screening and suspicious transaction flagging across high transaction volumes  ## Customization Replace OpenAI GPT-4 with Anthropic Claude or NVIDIA NIM in any agent node  ## Benefits Automates end-to-end fraud detection and compliance reporting, eliminating manual transaction reviews |
| Compliance Action Parser | outputParserStructured | Enforce structured compliance decision | — | Compliance Agent | ## Compliance Records Storage **What:** Airtable Tool logs all compliance findings and agent outputs throughout processing. **Why:** Maintains a real-time, audit-ready record of every transaction reviewed. |
| Investigation Agent Tool | agentTool | Tool: deep investigation | OpenAI Model - Investigation; Investigation Parser | Compliance Agent | ## Specialist Agent Processing **What:** Investigation, Risk Scoring, and Reporting agents run per risk routing path. **Why:** Each agent targets a distinct compliance function, improving detection accuracy. |
| OpenAI Model - Investigation | lmChatOpenAi | LLM for investigation tool | — | Investigation Agent Tool | ## Specialist Agent Processing **What:** Investigation, Risk Scoring, and Reporting agents run per risk routing path. **Why:** Each agent targets a distinct compliance function, improving detection accuracy. |
| Investigation Parser | outputParserStructured | Enforce structured investigation output | — | Investigation Agent Tool | ## Specialist Agent Processing **What:** Investigation, Risk Scoring, and Reporting agents run per risk routing path. **Why:** Each agent targets a distinct compliance function, improving detection accuracy. |
| Risk Scoring Agent Tool | agentTool | Tool: refined risk scoring | OpenAI Model - Risk Scoring; Risk Score Parser | Compliance Agent | ## Specialist Agent Processing **What:** Investigation, Risk Scoring, and Reporting agents run per risk routing path. **Why:** Each agent targets a distinct compliance function, improving detection accuracy. |
| OpenAI Model - Risk Scoring | lmChatOpenAi | LLM for risk scoring tool | — | Risk Scoring Agent Tool | ## Specialist Agent Processing **What:** Investigation, Risk Scoring, and Reporting agents run per risk routing path. **Why:** Each agent targets a distinct compliance function, improving detection accuracy. |
| Risk Score Parser | outputParserStructured | Enforce structured refined score output | — | Risk Scoring Agent Tool | ## Specialist Agent Processing **What:** Investigation, Risk Scoring, and Reporting agents run per risk routing path. **Why:** Each agent targets a distinct compliance function, improving detection accuracy. |
| Airtable Tool - Compliance Records | airtableTool | Tool: query/update compliance records | — | Compliance Agent | ## Compliance Records Storage **What:** Airtable Tool logs all compliance findings and agent outputs throughout processing. **Why:** Maintains a real-time, audit-ready record of every transaction reviewed. |
| Reporting Agent Tool | agentTool | Tool: generate compliance/SAR reports | OpenAI Model - Reporting; Report Parser | Compliance Agent | ## Specialist Agent Processing **What:** Investigation, Risk Scoring, and Reporting agents run per risk routing path. **Why:** Each agent targets a distinct compliance function, improving detection accuracy. |
| OpenAI Model - Reporting | lmChatOpenAi | LLM for reporting tool | — | Reporting Agent Tool | ## Specialist Agent Processing **What:** Investigation, Risk Scoring, and Reporting agents run per risk routing path. **Why:** Each agent targets a distinct compliance function, improving detection accuracy. |
| Report Parser | outputParserStructured | Enforce structured report output | — | Reporting Agent Tool | ## Specialist Agent Processing **What:** Investigation, Risk Scoring, and Reporting agents run per risk routing path. **Why:** Each agent targets a distinct compliance function, improving detection accuracy. |
| Merge Results | merge | Consolidate outputs | Compliance Agent (inputs 0/1/2) | Update Transaction Records | ## Merge & Update Records **What:** All agent outputs are consolidated and transaction records updated in Airtable. **Why:** Ensures a single, unified compliance status per transaction for downstream reporting. |
| Update Transaction Records | airtable | Persist final results to Airtable | Merge Results | — | ## Merge & Update Records **What:** All agent outputs are consolidated and transaction records updated in Airtable. **Why:** Ensures a single, unified compliance status per transaction for downstream reporting. |
| Sticky Note | stickyNote | Comment | — | — | ## Prerequisites n8n (cloud or self-hosted), OpenAI API key (GPT-4), Airtable account with configured base and appropriate table schema  ## Use Cases Compliance teams automating AML screening and suspicious transaction flagging across high transaction volumes  ## Customization Replace OpenAI GPT-4 with Anthropic Claude or NVIDIA NIM in any agent node  ## Benefits Automates end-to-end fraud detection and compliance reporting, eliminating manual transaction reviews |
| Sticky Note1 | stickyNote | Comment | — | — | ## Setup Steps  1. Import workflow JSON into your n8n instance. 2. Add OpenAI API credentials. 3. Set Schedule Trigger frequency aligned to your transaction processing cycle. 4. Update Workflow Configuration node with risk thresholds and compliance rule parameters. 5. Connect Airtable credentials and configure base/table IDs for Fetch Pending Transactions. |
| Sticky Note2 | stickyNote | Comment | — | — | ## How It Works  This workflow automates financial transaction monitoring, fraud detection, and regulatory compliance using OpenAI GPT-4 across coordinated specialist agents. It targets compliance officers, fraud analysts, and fintech operations teams managing high transaction volumes where manual review is too slow to catch emerging fraud patterns and compliance breaches in time. On schedule, the system fetches pending transactions from Airtable and routes them through a Transaction Signal Agent that classifies each by risk level—High, Medium, Low, or Unclassified. A Compliance Agent then coordinates three specialist agents: Investigation, Risk Scoring, and Reporting. Airtable stores all compliance records throughout. Results merge and update transaction records directly, giving compliance teams a fully automated, audit-ready pipeline that flags fraud, scores risk, and generates regulatory reports without manual intervention. |
| Sticky Note3 | stickyNote | Comment | — | — | ## Fetch Pending Transactions **What:** Retrieves unreviewed transaction records from Airtable. **Why:** Provides a structured, live transaction queue for AI risk analysis. |
| Sticky Note4 | stickyNote | Comment | — | — | ## Compliance Records Storage **What:** Airtable Tool logs all compliance findings and agent outputs throughout processing. **Why:** Maintains a real-time, audit-ready record of every transaction reviewed. |
| Sticky Note5 | stickyNote | Comment | — | — | ## Merge & Update Records **What:** All agent outputs are consolidated and transaction records updated in Airtable. **Why:** Ensures a single, unified compliance status per transaction for downstream reporting. |
| Sticky Note6 | stickyNote | Comment | — | — | ## Specialist Agent Processing **What:** Investigation, Risk Scoring, and Reporting agents run per risk routing path. **Why:** Each agent targets a distinct compliance function, improving detection accuracy. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name it: **AI-powered transaction fraud detection and compliance monitoring** (or your preferred name).
- Keep it **inactive** until credentials and Airtable IDs are set.

2) **Add “Schedule Trigger”**
- Node: **Schedule Trigger**
- Set interval: **Every 15 minutes**.
- Connect to **Workflow Configuration**.

3) **Add “Workflow Configuration” (Set node)**
- Node type: **Set**
- Add fields (and enable “Include Other Fields”):
  - `riskThresholdHigh` (Number) = `80`
  - `riskThresholdMedium` (Number) = `50`
  - `complianceFramework` (String) = `AML/KYC`
  - `investigationDepth` (String) = `comprehensive`
- Connect to **Fetch Pending Transactions**.

4) **Add “Fetch Pending Transactions” (Airtable Search)**
- Node type: **Airtable**
- Credentials: connect your **Airtable personal access token** (or OAuth, depending on your n8n setup).
- Configure:
  - Base: select your Base (or paste Base ID).
  - Table: select **Transactions** table (or paste Table ID).
  - Operation: **Search**
  - Filter by formula: `={Status} = "Pending"`
- Connect to **Transaction Signal Agent**.

5) **Add the Transaction Signal AI triplet**
- Add **OpenAI Model - Transaction Signal**
  - Node type: **OpenAI Chat Model (LangChain)**
  - Model: `gpt-4o`
  - Credential: **OpenAI API key** credential in n8n.
- Add **Transaction Signal Parser**
  - Node type: **Structured Output Parser**
  - Paste the schema (matching the workflow): includes `transactionId`, `validationStatus`, `riskScore`, `riskLevel`, `signals`, `anomalies`, `requiresInvestigation`, `reasoning`.
- Add **Transaction Signal Agent**
  - Node type: **AI Agent (LangChain)**
  - Prompt type: **Define**
  - Text: `{{ $json }}`
  - System message: transaction validation + signal extraction + preliminary risk scoring.
  - Enable output parsing.
- Wire:
  - OpenAI Model → Transaction Signal Agent (as **ai_languageModel**)
  - Transaction Signal Parser → Transaction Signal Agent (as **ai_outputParser**)
  - Fetch Pending Transactions → Transaction Signal Agent (main)

6) **Add “Route by Risk Level”**
- Node type: **Switch**
- Add 3 rules:
  - `{{$json.riskLevel}}` equals `high`
  - equals `medium`
  - equals `low`
- Set fallback output name to **Unclassified** (optional but recommended).
- Connect each of High/Medium/Low outputs to **Compliance Agent**.
- (Recommended) Also connect **Unclassified** to Compliance Agent or to a logging/exception path so items are not dropped.

7) **Add Compliance Agent core**
- Add **OpenAI Model - Compliance**
  - Model: `gpt-4o`
- Add **Compliance Action Parser**
  - Schema includes `transactionId`, `complianceAction`, `finalRiskScore`, `investigationSummary`, `complianceFrameworks[]`, `requiredActions[]`, `escalationRequired`, `reportGenerated`, `reasoning`.
- Add **Compliance Agent**
  - Text: `{{ $json }}`
  - System message: instruct orchestration across tools + Airtable + final decision.
  - Enable tool use and structured output.
- Wire:
  - OpenAI Model - Compliance → Compliance Agent (**ai_languageModel**)
  - Compliance Action Parser → Compliance Agent (**ai_outputParser**)
  - Route by Risk Level → Compliance Agent (main)

8) **Add tool: Airtable Tool - Compliance Records**
- Node type: **Airtable Tool**
- Configure Base + **Compliance Records** table.
- Operation: **Search** (as in the JSON).
- Connect Airtable Tool → Compliance Agent (**ai_tool**).
- (Recommended) If you truly need updates, add a second Airtable Tool node configured for **Update/Create** and connect it as well.

9) **Add tool: Investigation**
- Add **OpenAI Model - Investigation** (`gpt-4o`)
- Add **Investigation Parser** (schema with `investigationId`, `threatLevel`, etc.)
- Add **Investigation Agent Tool**
  - Input: `{{ $fromAI("transactionData", "...", "json") }}`
  - System message: deep investigation steps.
  - Enable output parser.
- Wire:
  - OpenAI Model - Investigation → Investigation Agent Tool (**ai_languageModel**)
  - Investigation Parser → Investigation Agent Tool (**ai_outputParser**)
  - Investigation Agent Tool → Compliance Agent (**ai_tool**)

10) **Add tool: Risk Scoring**
- Add **OpenAI Model - Risk Scoring** (`gpt-4o`)
- Add **Risk Score Parser**
- Add **Risk Scoring Agent Tool**
  - Input: `{{ $fromAI("transactionData", "...", "json") }}`
- Wire model + parser into the tool, then tool → Compliance Agent (**ai_tool**).

11) **Add tool: Reporting**
- Add **OpenAI Model - Reporting** (`gpt-4o`)
- Add **Report Parser**
- Add **Reporting Agent Tool**
  - Input: `{{ $fromAI("complianceData", "...", "json") }}`
- Wire model + parser into the tool, then tool → Compliance Agent (**ai_tool**).

12) **Add “Merge Results”**
- Node type: **Merge**
- Set **Number of Inputs = 3**
- Connect **Compliance Agent** output into Merge inputs 0/1/2 (to match the JSON).
- (Recommended) Replace this with a simpler flow (single merge or no merge) unless you truly have three distinct branches producing different payloads.

13) **Add “Update Transaction Records”**
- Node type: **Airtable**
- Operation: **Update**
- Configure Base + Transactions table.
- Map fields:
  - Status = `{{ $json.complianceAction }}`
  - Report ID = `{{ $json.reportId }}`
  - Risk Score = `{{ $json.finalRiskScore }}`
  - Updated At = `{{ $now.toISO() }}`
  - Compliance Action = `{{ $json.complianceAction }}`
  - Investigation Summary = `{{ $json.investigationSummary }}`
- **Critical:** ensure the node has the **Record ID** to update.
  - Typical approach: keep Airtable’s `id` from “Fetch Pending Transactions” and pass it through; then set Update “Record ID” to that `id`.
- Connect **Merge Results → Update Transaction Records**.

14) **Credentials setup**
- **OpenAI:** Add an OpenAI credential with API key; ensure the model `gpt-4o` is available for your account.
- **Airtable:** Add Airtable credentials and verify access to both the Transactions table and Compliance Records table.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Replace OpenAI GPT-4 with Anthropic Claude or NVIDIA NIM in any agent node” | Mentioned in prerequisites/customization sticky note (applies to all LangChain agent/model nodes). |
| Workflow processes pending transactions from Airtable, classifies risk (High/Medium/Low/Unclassified), then orchestrates Investigation/Risk/Reporting and writes results back for an audit-ready pipeline. | “How It Works” sticky note (overall design intent). |
| Ensure your Airtable base has an appropriate table schema (Status field, and fields used in Update). | Implied by prerequisites and Airtable nodes. |
| Potential design mismatches to validate: `reportId` used in Airtable update vs. not guaranteed by Compliance Agent schema; Merge with 3 inputs all from same node; Unclassified path not connected. | Derived from node configuration and connections. |
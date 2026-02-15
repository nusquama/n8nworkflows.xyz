Assess credential risk and route mitigation actions with GPT-4o-mini

https://n8nworkflows.xyz/workflows/assess-credential-risk-and-route-mitigation-actions-with-gpt-4o-mini-13331


# Assess credential risk and route mitigation actions with GPT-4o-mini

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title (given):** Assess credential risk and route mitigation actions with GPT-4o-mini  
**Workflow name (JSON):** AI-powered enterprise risk assessment & mitigation planning system

This workflow runs an **AI-orchestrated, multi-stage credential risk pipeline**: it generates (sample) credential records, has a **Coordination Agent** call three specialized AI tools in sequence (Validation → Verification → Risk Assessment), then **routes each credential by computed risk level** (critical/high/medium/low) to attach mitigation actions, and finally **merges all paths** into a unified reporting output.

### 1.1 Trigger & Global Configuration
Starts manually and sets workflow-wide thresholds (risk and verification thresholds).

### 1.2 Data Preparation (Sample Credentials)
Generates a list of structured credential items (passport, driver license, national ID, etc.) for testing.

### 1.3 Multi-Agent AI Orchestration (Validation → Verification → Risk)
A Coordination Agent uses GPT-4o-mini and calls three agent tools, each with structured output parsing.

### 1.4 Risk-Based Routing & Mitigation Actions
Switch routes results by `riskLevel`, then each branch sets escalation/automation instructions.

### 1.5 Consolidation & Final Reporting
All branches merge, and a final report wrapper is appended.

---

## 2. Block-by-Block Analysis

### Block 1 — Trigger & Workflow Configuration
**Overview:** Initiates execution and defines reusable numeric thresholds for validation/verification/risk categorization.

**Nodes involved:**
- Manual Trigger
- Workflow Configuration

#### Node: Manual Trigger
- **Type / role:** `n8n-nodes-base.manualTrigger` — entry point for on-demand runs.
- **Configuration:** No parameters.
- **Outputs:** One empty item to start the flow.
- **Connections:** → **Workflow Configuration**
- **Edge cases:** None; only runs when manually executed.

#### Node: Workflow Configuration
- **Type / role:** `n8n-nodes-base.set` — sets global config variables (thresholds).
- **Configuration choices (interpreted):**
  - Adds numeric fields:
    - `validationThresholdScore = 75`
    - `verificationRequiredScore = 60`
    - `criticalRiskThreshold = 80`
    - `highRiskThreshold = 60`
    - `mediumRiskThreshold = 40`
  - **Include other fields:** enabled (preserves incoming data).
- **Connections:** ← Manual Trigger; → Generate Sample Credential Data
- **Edge cases / failures:**
  - Mis-typed thresholds (string vs number) could break later comparisons if referenced (in this workflow, these thresholds are set but routing currently uses `riskLevel` string, not numeric thresholds).

---

### Block 2 — Sample Data Generation
**Overview:** Produces multiple credential records as individual n8n items for downstream AI processing.

**Nodes involved:**
- Generate Sample Credential Data

#### Node: Generate Sample Credential Data
- **Type / role:** `n8n-nodes-base.code` — generates test credentials array and returns as items.
- **Configuration choices:**
  - JavaScript builds 6 credential objects with fields like:
    - `credentialId`, `credentialType`, `holderName`, `issueDate`, `expiryDate`, `issuingAuthority`, `documentNumber`, `verificationStatus`, `metadata`.
  - Returns: `credentials.map(credential => ({ json: credential }))`
- **Connections:** ← Workflow Configuration; → Coordination Agent
- **Edge cases / failures:**
  - Code node runtime errors (syntax edits, missing return).
  - Date strings are assumed ISO-like; if non-ISO, the validation agent may flag.

---

### Block 3 — Multi-Agent AI Orchestration (LangChain in n8n)
**Overview:** A Coordination Agent orchestrates three specialized agent tools in strict sequence. Each tool uses GPT-4o-mini and a structured output parser to enforce JSON output contracts.

**Nodes involved:**
- OpenAI Model - Coordination Agent
- Coordination Agent
- OpenAI Model - Validation Agent
- Credential Validation Agent Tool
- Validation Output Parser
- OpenAI Model - Verification Agent
- Credential Verification Agent Tool
- Verification Output Parser
- OpenAI Model - Risk Assessment Agent
- Risk Assessment Agent Tool
- Risk Assessment Output Parser

#### Node: OpenAI Model - Coordination Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — chat LLM provider for the Coordination Agent.
- **Configuration choices:**
  - **Model:** `gpt-4o-mini`
  - Built-in tools: none configured
  - Uses **OpenAI credentials**: “OpenAi account”
- **Connections:** Provides `ai_languageModel` → Coordination Agent
- **Failure types:**
  - OpenAI auth/credential issues, quota exhaustion.
  - Model name availability changes.
  - Network timeouts.

#### Node: Coordination Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — orchestrator that calls agent tools.
- **Key configuration:**
  - **Prompt type:** “define”
  - **Input text:** `={{ $json.credentials }}`
    - **Important:** each incoming item from the Code node is a single credential object, not an object containing `credentials`. This expression likely evaluates to `undefined`.
  - **System message:** mandates sequential tool calls:
    1) Validation tool with credential data  
    2) Verification tool with validation results  
    3) Risk Assessment tool with verification results  
    Return only the final risk assessment.
- **Connections:**
  - Inputs:
    - `main` from Generate Sample Credential Data
    - `ai_languageModel` from OpenAI Model - Coordination Agent
    - `ai_tool` from the three Agent Tool nodes (they are “available tools” to the agent)
  - Outputs: `main` → Route by Risk Level
- **Edge cases / failures:**
  - **Likely data mapping bug:** `$json.credentials` should probably be `$json` (the credential record). If undefined, the agent may hallucinate or fail to call tools correctly.
  - Tool-call compliance: agent might not follow sequence under some conditions; system prompt reduces but does not eliminate this risk.
  - If structured parsing fails in downstream tools, agent tool execution fails.

**Recommended fix:** change agent input to `={{ $json }}` or set a field `credentials` upstream.

#### Node: OpenAI Model - Validation Agent
- **Type / role:** Chat LLM provider for validation tool.
- **Model:** `gpt-4o-mini`
- **Connections:** `ai_languageModel` → Credential Validation Agent Tool
- **Failures:** same OpenAI failure modes.

#### Node: Credential Validation Agent Tool
- **Type / role:** `@n8n/n8n-nodes-langchain.agentTool` — tool callable by the Coordination Agent; validates structure/format/security features.
- **Key configuration:**
  - **Tool input text expression:**  
    `={{ $fromAI("credentialData", "Credential data to validate", "json") }}`
  - **System message:** detailed validation rules, scoring, required fields, must return structured JSON.
  - **Has output parser:** enabled.
- **Connections:**
  - Tool is exposed to **Coordination Agent** via `ai_tool`.
  - Receives output parser via `ai_outputParser` from Validation Output Parser.
- **Edge cases / failures:**
  - If Coordination Agent does not provide `credentialData` in the expected shape, parsing/validation will degrade.
  - LLM may output non-JSON; parser will fail.

#### Node: Validation Output Parser
- **Type / role:** `outputParserStructured` — enforces JSON schema for validation result.
- **Schema highlights:**
  - `credentialId: string`
  - `isValid: boolean`
  - `validationScore: 0..100`
  - `issues[]` with `{severity, description}`
  - `expiryStatus: valid|expired|expiring_soon`
  - plus booleans for integrity/compliance/security features and `reasoning`
- **Connections:** `ai_outputParser` → Credential Validation Agent Tool
- **Failure types:**
  - Schema mismatch (missing fields, wrong types).
  - LLM returns extra text; depending on parser strictness, may fail.

#### Node: OpenAI Model - Verification Agent
- **Type / role:** Chat LLM provider for verification tool.
- **Model:** `gpt-4o-mini`
- **Connections:** `ai_languageModel` → Credential Verification Agent Tool

#### Node: Credential Verification Agent Tool
- **Type / role:** agentTool — verifies authenticity, cross-references, assesses anomalies.
- **Tool input text expression:**  
  `={{ $fromAI("validationResults", "Validation results from validation agent", "json") }}`
- **System message:** requires structured output, scores, recommended action.
- **Has output parser:** enabled.
- **Connections:**
  - Exposed to Coordination Agent via `ai_tool`
  - Parser provided by Verification Output Parser
- **Edge cases:**
  - If validation output is missing required properties, verification reasoning may be inconsistent or parser may fail.

#### Node: Verification Output Parser
- **Type / role:** structured parser for verification.
- **Schema highlights (required):**
  - `verificationStatus: verified|failed|pending_review`
  - `verificationScore: 0..100`
  - `crossReferenceResults: { databaseMatch, authorityValidation }`
  - `biometricVerification: boolean`
  - `documentAuthenticity: authentic|suspicious|counterfeit`
  - `anomaliesDetected: string[]`
  - `recommendedAction`, `reasoning`
- **Connections:** `ai_outputParser` → Credential Verification Agent Tool
- **Failures:** schema mismatch.

#### Node: OpenAI Model - Risk Assessment Agent
- **Type / role:** Chat LLM provider for risk assessment tool.
- **Model:** `gpt-4o-mini`
- **Connections:** `ai_languageModel` → Risk Assessment Agent Tool

#### Node: Risk Assessment Agent Tool
- **Type / role:** agentTool — turns verification output into risk scoring and mitigations.
- **Tool input text expression:**  
  `={{ $fromAI("verificationResults", "Verification results from verification agent", "json") }}`
- **System message:** defines risk classification and additive risk factor model; must output structured JSON.
- **Has output parser:** enabled.
- **Connections:**
  - Exposed to Coordination Agent via `ai_tool`
  - Parser via Risk Assessment Output Parser
- **Edge cases:**
  - If anomalies array is large, risk score could exceed bounds; schema constrains 0..100, so the model must normalize.

#### Node: Risk Assessment Output Parser
- **Type / role:** structured parser for risk assessment.
- **Schema highlights:**
  - `riskLevel: critical|high|medium|low`
  - `riskScore: 0..100`
  - `fraudProbability: 0..100`
  - `identityTheftRisk: boolean`
  - `complianceViolations: string[]`
  - `geopoliticalRisk: high|medium|low|none`
  - `recommendedEscalation: boolean`
  - `mitigationActions: string[]`
  - `reasoning: string`
- **Connections:** `ai_outputParser` → Risk Assessment Agent Tool
- **Failures:** schema mismatch.

---

### Block 4 — Risk Routing & Branch Processing
**Overview:** Routes each processed credential by `riskLevel` and attaches operational handling metadata.

**Nodes involved:**
- Route by Risk Level
- Process Critical Risk
- Process High Risk
- Process Medium Risk
- Process Low Risk

#### Node: Route by Risk Level
- **Type / role:** `n8n-nodes-base.switch` — conditional branching.
- **Key expressions:**
  - Compares `={{ $json.output.riskLevel }}` to `critical|high|medium|low`
  - Produces 4 named outputs: Critical, High, Medium, Low (renamed outputs enabled).
- **Connections:** ← Coordination Agent; → each Process * Risk node
- **Edge cases / failures:**
  - The path expects `riskLevel` at `$json.output.riskLevel`.
    - Depending on how the LangChain Agent node structures results, the risk assessment may be at `$json` or `$json.output` or another field. If mismatch, none of the rules match and items are dropped.
  - Missing/invalid `riskLevel` causes no branch match.

#### Node: Process Critical Risk
- **Type / role:** Set node — assigns escalation instructions for critical.
- **Adds fields:**
  - `processingAction = IMMEDIATE_ESCALATION`
  - `notificationRequired = true`
  - `escalationTeam = Security & Compliance`
  - `priority = P0 - Critical`
  - `automatedActions = "Freeze credential, Alert security team, Initiate investigation"` (string)
- **Connections:** → Merge All Risk Paths (input 0)
- **Edge cases:**
  - `automatedActions` type differs from High/Medium (arrays there). Downstream consumers should handle both string/array.

#### Node: Process High Risk
- **Type / role:** Set node — assigns high-risk handling.
- **Adds fields:**
  - `processingAction = ESCALATION_RECOMMENDED`
  - `notificationRequired = true`
  - `escalationTeam = Verification Team`
  - `priority = P1 - High`
  - `automatedActions = ["Flag for manual review","Enhanced monitoring"]` (array)
- **Connections:** → Merge All Risk Paths (input 1)

#### Node: Process Medium Risk
- **Type / role:** Set node — assigns medium-risk handling.
- **Adds fields:**
  - `processingAction = ENHANCED_MONITORING`
  - `notificationRequired = false`
  - `escalationTeam = Operations Team`
  - `priority = P2 - Medium`
  - `automatedActions = ["Additional verification checks","Periodic review"]` (array)
- **Connections:** → Merge All Risk Paths (input 2)

#### Node: Process Low Risk
- **Type / role:** Set node — assigns low-risk handling.
- **Adds fields:**
  - `processingAction = STANDARD_PROCESSING`
  - `notificationRequired = false`
  - `escalationTeam = None`
  - `priority = P3 - Low`
  - `automatedActions = "Approve for standard processing"` (string)
- **Connections:** → Merge All Risk Paths (input 3)

---

### Block 5 — Consolidation & Reporting
**Overview:** Merges all routed branches into a single stream and appends final reporting metadata.

**Nodes involved:**
- Merge All Risk Paths
- Final Report

#### Node: Merge All Risk Paths
- **Type / role:** `n8n-nodes-base.merge` — merges 4 inputs into one output.
- **Configuration:** `numberInputs = 4`
- **Connections:** receives from all four Process nodes; outputs → Final Report
- **Edge cases / failures:**
  - Merge mode defaults matter; with multiple inputs, n8n merge behavior can be “append” style depending on node version/settings. Here it’s effectively used to recombine the split routes.
  - If some risk levels produce zero items, merge still works but may wait depending on merge configuration; in practice, with routing, it usually passes through items from whichever input has data.

#### Node: Final Report
- **Type / role:** Set node — attaches final report metadata per item.
- **Key assignments:**
  - `reportGeneratedAt = {{$now.toISO()}}`
  - `workflowStatus = "Completed"`
  - `totalCredentialsProcessed = {{ $json.credentialId ? 1 : 0 }}`
  - `finalRecommendation = {{ $json.processingAction }}`
- **Connections:** terminal node (no outputs).
- **Edge cases:**
  - `totalCredentialsProcessed` is computed **per item** (always 1 if `credentialId` exists). It is not a true aggregate count of all credentials. If you need totals, add an Item Lists / Aggregate step after merge.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Manual Trigger | Manual Trigger | Start workflow manually | — | Workflow Configuration |  |
| Workflow Configuration | Set | Define global thresholds | Manual Trigger | Generate Sample Credential Data |  |
| Generate Sample Credential Data | Code | Create sample credential items | Workflow Configuration | Coordination Agent | ## Flexible Assessment & Multi-Agent Risk Analysis  \n**What**: Coordination Agent orchestrates three specialized agents for credential validation, verification, and risk scoring  \n**Why**: Parallel expert evaluation identifies risks across identity, data integrity, and threat dimensions simultaneously |
| OpenAI Model - Coordination Agent | OpenAI Chat Model (LangChain) | LLM backend for orchestration agent | — | Coordination Agent (ai_languageModel) | ## Flexible Assessment & Multi-Agent Risk Analysis  \n**What**: Coordination Agent orchestrates three specialized agents for credential validation, verification, and risk scoring  \n**Why**: Parallel expert evaluation identifies risks across identity, data integrity, and threat dimensions simultaneously |
| Coordination Agent | LangChain Agent | Orchestrates tool calls Validation→Verification→Risk | Generate Sample Credential Data; Agent Tools; OpenAI Model - Coordination Agent | Route by Risk Level | ## Flexible Assessment & Multi-Agent Risk Analysis  \n**What**: Coordination Agent orchestrates three specialized agents for credential validation, verification, and risk scoring  \n**Why**: Parallel expert evaluation identifies risks across identity, data integrity, and threat dimensions simultaneously |
| OpenAI Model - Validation Agent | OpenAI Chat Model (LangChain) | LLM backend for validation tool | — | Credential Validation Agent Tool (ai_languageModel) | ## Flexible Assessment & Multi-Agent Risk Analysis  \n**What**: Coordination Agent orchestrates three specialized agents for credential validation, verification, and risk scoring  \n**Why**: Parallel expert evaluation identifies risks across identity, data integrity, and threat dimensions simultaneously |
| Credential Validation Agent Tool | LangChain Agent Tool | Validate structure/format/expiry/security features | Coordination Agent (tool invocation) | Coordination Agent (tool result) | ## Flexible Assessment & Multi-Agent Risk Analysis  \n**What**: Coordination Agent orchestrates three specialized agents for credential validation, verification, and risk scoring  \n**Why**: Parallel expert evaluation identifies risks across identity, data integrity, and threat dimensions simultaneously |
| Validation Output Parser | Structured Output Parser (LangChain) | Enforce JSON schema for validation output | — | Credential Validation Agent Tool (ai_outputParser) | ## Flexible Assessment & Multi-Agent Risk Analysis  \n**What**: Coordination Agent orchestrates three specialized agents for credential validation, verification, and risk scoring  \n**Why**: Parallel expert evaluation identifies risks across identity, data integrity, and threat dimensions simultaneously |
| OpenAI Model - Verification Agent | OpenAI Chat Model (LangChain) | LLM backend for verification tool | — | Credential Verification Agent Tool (ai_languageModel) | ## Flexible Assessment & Multi-Agent Risk Analysis  \n**What**: Coordination Agent orchestrates three specialized agents for credential validation, verification, and risk scoring  \n**Why**: Parallel expert evaluation identifies risks across identity, data integrity, and threat dimensions simultaneously |
| Credential Verification Agent Tool | LangChain Agent Tool | Verify authenticity / anomalies / cross-reference | Coordination Agent (tool invocation) | Coordination Agent (tool result) | ## Flexible Assessment & Multi-Agent Risk Analysis  \n**What**: Coordination Agent orchestrates three specialized agents for credential validation, verification, and risk scoring  \n**Why**: Parallel expert evaluation identifies risks across identity, data integrity, and threat dimensions simultaneously |
| Verification Output Parser | Structured Output Parser (LangChain) | Enforce JSON schema for verification output | — | Credential Verification Agent Tool (ai_outputParser) | ## Flexible Assessment & Multi-Agent Risk Analysis  \n**What**: Coordination Agent orchestrates three specialized agents for credential validation, verification, and risk scoring  \n**Why**: Parallel expert evaluation identifies risks across identity, data integrity, and threat dimensions simultaneously |
| OpenAI Model - Risk Assessment Agent | OpenAI Chat Model (LangChain) | LLM backend for risk tool | — | Risk Assessment Agent Tool (ai_languageModel) | ## Flexible Assessment & Multi-Agent Risk Analysis  \n**What**: Coordination Agent orchestrates three specialized agents for credential validation, verification, and risk scoring  \n**Why**: Parallel expert evaluation identifies risks across identity, data integrity, and threat dimensions simultaneously |
| Risk Assessment Agent Tool | LangChain Agent Tool | Compute risk level/score & mitigations | Coordination Agent (tool invocation) | Coordination Agent (tool result) | ## Flexible Assessment & Multi-Agent Risk Analysis  \n**What**: Coordination Agent orchestrates three specialized agents for credential validation, verification, and risk scoring  \n**Why**: Parallel expert evaluation identifies risks across identity, data integrity, and threat dimensions simultaneously |
| Risk Assessment Output Parser | Structured Output Parser (LangChain) | Enforce JSON schema for risk output | — | Risk Assessment Agent Tool (ai_outputParser) | ## Flexible Assessment & Multi-Agent Risk Analysis  \n**What**: Coordination Agent orchestrates three specialized agents for credential validation, verification, and risk scoring  \n**Why**: Parallel expert evaluation identifies risks across identity, data integrity, and threat dimensions simultaneously |
| Route by Risk Level | Switch | Branch by `riskLevel` | Coordination Agent | Process Critical Risk; Process High Risk; Process Medium Risk; Process Low Risk | ## Priority-Based Routing  \n**What**: Routes findings by severity—critical/high/medium/low through conditional logic for appropriate handling  \n**Why**: Risk stratification ensures urgent threats receive immediate attention while maintaining complete documentation |
| Process Critical Risk | Set | Attach critical escalation actions | Route by Risk Level (Critical) | Merge All Risk Paths | ## Priority-Based Routing  \n**What**: Routes findings by severity—critical/high/medium/low through conditional logic for appropriate handling  \n**Why**: Risk stratification ensures urgent threats receive immediate attention while maintaining complete documentation |
| Process High Risk | Set | Attach high-risk escalation actions | Route by Risk Level (High) | Merge All Risk Paths | ## Priority-Based Routing  \n**What**: Routes findings by severity—critical/high/medium/low through conditional logic for appropriate handling  \n**Why**: Risk stratification ensures urgent threats receive immediate attention while maintaining complete documentation |
| Process Medium Risk | Set | Attach monitoring actions | Route by Risk Level (Medium) | Merge All Risk Paths | ## Priority-Based Routing  \n**What**: Routes findings by severity—critical/high/medium/low through conditional logic for appropriate handling  \n**Why**: Risk stratification ensures urgent threats receive immediate attention while maintaining complete documentation |
| Process Low Risk | Set | Attach standard processing actions | Route by Risk Level (Low) | Merge All Risk Paths | ## Priority-Based Routing  \n**What**: Routes findings by severity—critical/high/medium/low through conditional logic for appropriate handling  \n**Why**: Risk stratification ensures urgent threats receive immediate attention while maintaining complete documentation |
| Merge All Risk Paths | Merge | Recombine branched results | Process Critical Risk; Process High Risk; Process Medium Risk; Process Low Risk | Final Report | ## Unified Report Generation  \n**What**: Merges all risk assessments into consolidated report with prioritized recommendations  \n**Why**: Single comprehensive view enables executive decision-making and coordinated mitigation planning across organization |
| Final Report | Set | Add final report metadata | Merge All Risk Paths | — | ## Unified Report Generation  \n**What**: Merges all risk assessments into consolidated report with prioritized recommendations  \n**Why**: Single comprehensive view enables executive decision-making and coordinated mitigation planning across organization |
| Sticky Note | Sticky Note | Documentation/commentary | — | — | ## How It Works\nThis workflow automates comprehensive enterprise risk assessment and mitigation planning for organizations managing complex operational, financial, and compliance risks. Designed for risk managers, internal audit teams, and executive leadership, it solves the challenge of continuously evaluating multi-dimensional risks, validating threat severity, and coordinating appropriate mitigation strategies across diverse business functions. The system triggers on-demand or scheduled assessments, generates sample credential data for testing, deploys a Coordination Agent to orchestrate specialized risk evaluations through parallel AI agents (Credential Validation verifies identity risks, Credential Verification confirms data accuracy, Risk Assessment evaluates threat levels), routes findings by severity (critical/high/medium/low), and merges outputs into consolidated reports. By combining multi-agent risk analysis with intelligent prioritization and unified reporting, organizations achieve 360-degree risk visibility, reduce assessment cycles from weeks to hours, ensure consistent evaluation frameworks, and enable proactive mitigation before risks materialize into losses. |
| Sticky Note1 | Sticky Note | Setup guidance | — | — | ## Setup Steps\n1. Connect **Manual Trigger** for on-demand assessments or configure **Schedule Trigger** for routine evaluations\n2. Configure **risk data sources** \n3. Add **AI model API keys** to Coordination Agent and all specialized agents\n4. Define **risk scoring criteria** and severity thresholds in agent prompts aligned with company risk appetite\n5. Configure **routing conditions** for each risk level with appropriate handling workflows\n6. Set up **reporting output** format and distribution channels for consolidated risk reports |
| Sticky Note2 | Sticky Note | Prerequisites/use cases/benefits | — | — | ## Prerequisites\nEnterprise risk management system access, AI service accounts\n## Use Cases\nCybersecurity risk assessments, fraud risk evaluations, third-party vendor risk reviews\n## Customization\nModify agent prompts for industry-specific risk frameworks (NIST, ISO 31000, COSO)\n## Benefits\nReduces risk assessment time from weeks to hours, provides 360-degree risk visibility |
| Sticky Note3 | Sticky Note | Reporting intent | — | — | ## Unified Report Generation\n**What**: Merges all risk assessments into consolidated report with prioritized recommendations  \n**Why**: Single comprehensive view enables executive decision-making and coordinated mitigation planning across organization |
| Sticky Note4 | Sticky Note | Routing intent | — | — | ## Priority-Based Routing\n**What**: Routes findings by severity—critical/high/medium/low through conditional logic for appropriate handling  \n**Why**: Risk stratification ensures urgent threats receive immediate attention while maintaining complete documentation |
| Sticky Note5 | Sticky Note | Multi-agent intent | — | — | ## Flexible Assessment & Multi-Agent Risk Analysis\n**What**: Coordination Agent orchestrates three specialized agents for credential validation, verification, and risk scoring  \n**Why**: Parallel expert evaluation identifies risks across identity, data integrity, and threat dimensions simultaneously |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** in n8n.

2) **Add “Manual Trigger”** (`Manual Trigger` node).
   - No configuration needed.

3) **Add “Set” node** named **Workflow Configuration**.
   - Add number fields:
     - `validationThresholdScore` = 75
     - `verificationRequiredScore` = 60
     - `criticalRiskThreshold` = 80
     - `highRiskThreshold` = 60
     - `mediumRiskThreshold` = 40
   - Enable **Include Other Fields**.
   - Connect: **Manual Trigger → Workflow Configuration**

4) **Add “Code” node** named **Generate Sample Credential Data**.
   - Paste JS that returns multiple items (each item is one credential object). Ensure it ends with:
     - `return credentials.map(credential => ({ json: credential }));`
   - Connect: **Workflow Configuration → Generate Sample Credential Data**

5) **Create OpenAI credentials** (in n8n Credentials):
   - Add **OpenAI API** credential (API key).
   - Ensure the n8n instance has access to `@n8n/n8n-nodes-langchain` nodes.

6) **Add OpenAI Chat Model node** named **OpenAI Model - Coordination Agent**.
   - Node type: **OpenAI Chat Model (LangChain)** (`lmChatOpenAi`)
   - Model: `gpt-4o-mini`
   - Select your OpenAI credential.

7) **Add LangChain Agent node** named **Coordination Agent**.
   - Prompt type: **Define**
   - **System message:** use the orchestration instructions (Validation → Verification → Risk → return final risk only).
   - **Input text:** set to the credential item.
     - Recommended: `={{ $json }}` (to avoid `$json.credentials` being undefined).
   - Connect:
     - **Generate Sample Credential Data (main) → Coordination Agent (main)**
     - **OpenAI Model - Coordination Agent (ai_languageModel) → Coordination Agent (ai_languageModel)**

8) **Validation tool setup**
   1. Add OpenAI Chat Model node **OpenAI Model - Validation Agent**, model `gpt-4o-mini`, same credential.
   2. Add **Structured Output Parser** node **Validation Output Parser** with the validation JSON schema (credentialId, isValid, validationScore, issues, etc.).
   3. Add **Agent Tool** node **Credential Validation Agent Tool**:
      - Tool description: “Validates credential data structure, format compliance, and data integrity”
      - System message: validation specialist instructions and scoring bands.
      - Tool input expression: `={{ $fromAI("credentialData", "Credential data to validate", "json") }}`
      - Enable **Has Output Parser**
   4. Connect:
      - **OpenAI Model - Validation Agent (ai_languageModel) → Credential Validation Agent Tool (ai_languageModel)**
      - **Validation Output Parser (ai_outputParser) → Credential Validation Agent Tool (ai_outputParser)**
      - **Credential Validation Agent Tool (ai_tool) → Coordination Agent (ai_tool)**

9) **Verification tool setup**
   1. Add OpenAI Chat Model node **OpenAI Model - Verification Agent** (`gpt-4o-mini`).
   2. Add **Structured Output Parser** node **Verification Output Parser** with the verification schema.
   3. Add **Agent Tool** node **Credential Verification Agent Tool**:
      - Tool input expression: `={{ $fromAI("validationResults", "Validation results from validation agent", "json") }}`
      - Enable output parser
      - System message: verification instructions, scoring, required fields.
   4. Connect:
      - **OpenAI Model - Verification Agent → Credential Verification Agent Tool (ai_languageModel)**
      - **Verification Output Parser → Credential Verification Agent Tool (ai_outputParser)**
      - **Credential Verification Agent Tool (ai_tool) → Coordination Agent (ai_tool)**

10) **Risk assessment tool setup**
   1. Add OpenAI Chat Model node **OpenAI Model - Risk Assessment Agent** (`gpt-4o-mini`).
   2. Add **Structured Output Parser** node **Risk Assessment Output Parser** with the risk schema.
   3. Add **Agent Tool** node **Risk Assessment Agent Tool**:
      - Tool input expression: `={{ $fromAI("verificationResults", "Verification results from verification agent", "json") }}`
      - Enable output parser
      - System message: risk scoring rules and classification bands.
   4. Connect:
      - **OpenAI Model - Risk Assessment Agent → Risk Assessment Agent Tool (ai_languageModel)**
      - **Risk Assessment Output Parser → Risk Assessment Agent Tool (ai_outputParser)**
      - **Risk Assessment Agent Tool (ai_tool) → Coordination Agent (ai_tool)**

11) **Add Switch node** named **Route by Risk Level**.
   - Create 4 rules (equals):
     - `={{ $json.output.riskLevel }}` equals `critical`
     - equals `high`
     - equals `medium`
     - equals `low`
   - Connect: **Coordination Agent → Route by Risk Level**
   - If your agent output is not under `$json.output`, adjust to where `riskLevel` actually appears (common alternatives: `$json.riskLevel`, `$json.result.riskLevel`).

12) **Add four “Set” nodes**:
   - **Process Critical Risk** (P0, immediate escalation, notification true, etc.)
   - **Process High Risk** (P1, notification true, actions array)
   - **Process Medium Risk** (P2, monitoring)
   - **Process Low Risk** (P3, standard)
   - Connect each Switch output to its corresponding Set node.

13) **Add Merge node** named **Merge All Risk Paths**.
   - Set **Number of Inputs = 4**
   - Connect:
     - Process Critical Risk → Merge (input 0)
     - Process High Risk → Merge (input 1)
     - Process Medium Risk → Merge (input 2)
     - Process Low Risk → Merge (input 3)

14) **Add Set node** named **Final Report**.
   - Fields:
     - `reportGeneratedAt = {{ $now.toISO() }}`
     - `workflowStatus = Completed`
     - `totalCredentialsProcessed = {{ $json.credentialId ? 1 : 0 }}`
     - `finalRecommendation = {{ $json.processingAction }}`
   - Connect: **Merge All Risk Paths → Final Report**

15) **Run the workflow manually** and verify:
   - Coordination Agent successfully calls all three tools.
   - Parser nodes accept outputs (no schema errors).
   - Switch routes items to the expected branches.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The Coordination Agent input expression in the JSON is `{{$json.credentials}}`, but the sample generator outputs credentials as individual items (`$json` is the credential). Consider changing Coordination Agent input to `{{$json}}` or altering upstream data shape. | Prevents undefined input to the orchestration agent |
| Routing uses `{{$json.output.riskLevel}}`. Confirm the actual output shape of the LangChain Agent node in your n8n version; adjust the Switch expression accordingly. | Avoids “no rule matched” drops |
| `automatedActions` is a string in Critical/Low but an array in High/Medium; standardize type if downstream systems require consistent schema. | Downstream integration robustness |
| “totalCredentialsProcessed” is computed per item (1/0), not aggregated across all credentials. Use an aggregate step if you need a true total. | Reporting accuracy |
| Sticky notes describe intended enterprise risk use cases and customization for frameworks (NIST, ISO 31000, COSO). | Design intent / prompt customization guidance |
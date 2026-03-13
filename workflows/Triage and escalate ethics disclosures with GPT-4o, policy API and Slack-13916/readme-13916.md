Triage and escalate ethics disclosures with GPT-4o, policy API and Slack

https://n8nworkflows.xyz/workflows/triage-and-escalate-ethics-disclosures-with-gpt-4o--policy-api-and-slack-13916


# Triage and escalate ethics disclosures with GPT-4o, policy API and Slack

# 1. Workflow Overview

This workflow receives ethics disclosures via HTTP, analyzes them with a central AI governance agent supported by multiple specialized AI tools, routes the case by risk level, stores the outcome, alerts oversight teams for critical cases, and returns a structured response to the caller.

Its main use case is automated governance intake for employee or stakeholder ethics disclosures in regulated or policy-sensitive environments. It is designed to reduce manual triage, increase consistency, preserve explainability, and maintain auditable case handling.

## 1.1 Input Reception and Case Intake
The workflow starts with a webhook that accepts POST requests containing disclosure data. The webhook does not respond immediately; instead, it waits for a later `Respond to Webhook` node.

## 1.2 Central Governance AI Orchestration
A central Governance Agent receives the disclosure body and coordinates analysis. It uses:
- a primary GPT-4o model,
- buffer memory,
- four specialized AI agent tools,
- a policy API tool,
- an audit trail storage tool,
- a Slack notification tool,
- and a structured output parser.

## 1.3 Structured Decisioning and Risk Routing
The Governance Agent returns a structured decision containing fields such as `riskLevel`, `decision`, `investigationRequired`, and `nextActions`. A Switch node routes the case into critical/high versus medium/low handling branches.

## 1.4 Critical Case Handling
Critical and high-risk cases are normalized into a standard record, stored in a dedicated data table, and sent as a Slack alert to an oversight team.

## 1.5 Standard Case Handling
Medium and low-risk cases are normalized and stored in a separate data table without the critical Slack escalation step.

## 1.6 Merge and Final API Response
Both outcome paths converge in a Merge node, after which the workflow returns a JSON response confirming that the disclosure was processed.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception

### Overview
This block exposes an HTTP endpoint for incoming ethics disclosures. It is the workflow entry point and passes the received payload to the AI orchestration block.

### Nodes Involved
- Ethics Disclosure Webhook

### Node Details

#### Ethics Disclosure Webhook
- **Type and technical role:** `n8n-nodes-base.webhook`; receives inbound HTTP requests.
- **Configuration choices:**
  - HTTP method: `POST`
  - Path: `ethics-disclosure`
  - Response mode: `responseNode`, meaning the reply is deferred until `Send Response`
- **Key expressions or variables used:**
  - Downstream nodes reference payload data through `{{ $json.body }}` or nested fields like `{{ $json.body.caseId }}`
- **Input and output connections:**
  - No input; this is the trigger
  - Outputs to `Governance Agent`
- **Version-specific requirements:**
  - Uses type version `2.1`
  - Requires an n8n instance reachable by the client posting disclosures
- **Edge cases or potential failure types:**
  - Invalid HTTP method
  - Missing or malformed JSON body
  - Caller timeout if downstream execution is long
  - No built-in authentication configured in the node itself, so endpoint hardening must be done separately if exposed publicly
- **Sub-workflow reference:** None

---

## 2.2 Central Governance AI Orchestration

### Overview
This is the core intelligence layer. The Governance Agent receives the disclosure text, can call specialized agent tools and utility tools, and is forced to emit a structured output through a schema-based parser.

### Nodes Involved
- Governance Agent
- Governance Model
- Governance Memory
- Governance Output Parser
- Ethics Monitoring Agent Tool
- Ethics Monitoring Model
- Investigation Agent Tool
- Investigation Model
- Reporting Agent Tool
- Reporting Model
- Escalation Agent Tool
- Escalation Model
- Audit Trail Storage Tool
- Policy Database API Tool
- Slack Notification Tool

### Node Details

#### Governance Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`; orchestrating LLM agent.
- **Configuration choices:**
  - Input text: `{{ $json.body }}`
  - Has structured output parser enabled
  - System prompt defines it as a governance orchestrator responsible for ethics monitoring, investigations, reporting, escalation, explainability, and human checkpoints
- **Key expressions or variables used:**
  - `{{ $json.body }}`
- **Input and output connections:**
  - Main input from `Ethics Disclosure Webhook`
  - AI language model input from `Governance Model`
  - AI memory input from `Governance Memory`
  - AI tool inputs from all specialized tool nodes
  - AI output parser input from `Governance Output Parser`
  - Main output to `Risk Level Router`
- **Version-specific requirements:**
  - Uses type version `3.1`
  - Requires compatible LangChain-capable n8n version with agent/tool ports
- **Edge cases or potential failure types:**
  - LLM output may fail schema validation if the structured result is incomplete or malformed
  - Tool invocation may fail if downstream credentials or placeholders are not configured
  - If `$json.body` is not a string-like structure, model behavior may become inconsistent
  - Long-running tool usage may delay the webhook response
- **Sub-workflow reference:** None

#### Governance Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; primary chat model for the Governance Agent.
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0.2` for relatively stable decisioning
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Connected to `Governance Agent` via `ai_languageModel`
- **Version-specific requirements:**
  - Uses type version `1.3`
  - Requires valid OpenAI credentials
- **Edge cases or potential failure types:**
  - OpenAI authentication failure
  - Rate limits or quota exhaustion
  - Model availability changes
- **Sub-workflow reference:** None

#### Governance Memory
- **Type and technical role:** `@n8n/n8n-nodes-langchain.memoryBufferWindow`; conversational memory for the Governance Agent.
- **Configuration choices:**
  - Default settings; no explicit window parameters set
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Connected to `Governance Agent` via `ai_memory`
- **Version-specific requirements:**
  - Uses type version `1.3`
- **Edge cases or potential failure types:**
  - Limited practical benefit if each webhook execution is stateless and isolated
  - Behavior depends on n8n agent memory implementation and execution scope
- **Sub-workflow reference:** None

#### Governance Output Parser
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`; enforces a JSON schema on the Governance Agent output.
- **Configuration choices:**
  - Manual JSON schema with fields:
    - `decision` string
    - `riskLevel` enum: `low`, `medium`, `high`, `critical`
    - `investigationRequired` boolean
    - `escalationPriority` string
    - `reasoning` string
    - `nextActions` array of strings
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Connected to `Governance Agent` via `ai_outputParser`
- **Version-specific requirements:**
  - Uses type version `1.3`
- **Edge cases or potential failure types:**
  - Parser failure if agent returns invalid JSON or missing keys
  - Routing failure later if `riskLevel` is not one of the four expected values
- **Sub-workflow reference:** None

#### Ethics Monitoring Agent Tool
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agentTool`; specialist AI tool callable by the Governance Agent.
- **Configuration choices:**
  - Tool input text sourced with `$fromAI('disclosure_data', ...)`
  - System message focuses on conflicts of interest, policy violations, and compliance risks
  - Tool description explains that it evaluates disclosures and returns risk/compliance assessments
- **Key expressions or variables used:**
  - `{{ $fromAI('disclosure_data', 'The disclosure data to evaluate including conflicts, declarations, and policy signals') }}`
- **Input and output connections:**
  - Receives model input from `Ethics Monitoring Model`
  - Exposed to `Governance Agent` via `ai_tool`
- **Version-specific requirements:**
  - Uses type version `3`
  - Requires agent-tool compatible n8n version
- **Edge cases or potential failure types:**
  - Tool input may be vague or malformed if the governing agent calls it poorly
  - LLM hallucination risk if no structured disclosure schema is enforced at intake
- **Sub-workflow reference:** None

#### Ethics Monitoring Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; model backing the Ethics Monitoring tool.
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0.1`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Connected to `Ethics Monitoring Agent Tool`
- **Version-specific requirements:**
  - Uses type version `1.3`
  - Requires OpenAI credentials
- **Edge cases or potential failure types:**
  - Standard OpenAI errors: auth, quota, timeouts, rate limits
- **Sub-workflow reference:** None

#### Investigation Agent Tool
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agentTool`; specialist investigation tool.
- **Configuration choices:**
  - Input via `$fromAI('investigation_request', ...)`
  - System message defines evidence collection, interviews, timeline reconstruction, impact, and mitigation recommendations
- **Key expressions or variables used:**
  - `{{ $fromAI('investigation_request', 'The investigation request with case details, flagged issues, and scope') }}`
- **Input and output connections:**
  - Receives model input from `Investigation Model`
  - Exposed to `Governance Agent`
- **Version-specific requirements:**
  - Uses type version `3`
- **Edge cases or potential failure types:**
  - Can produce procedural-sounding output without real evidence if no external evidence source exists
  - May be invoked unnecessarily if the main agent overuses tools
- **Sub-workflow reference:** None

#### Investigation Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; model for the investigation tool.
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0.2`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Connected to `Investigation Agent Tool`
- **Version-specific requirements:**
  - Uses type version `1.3`
  - Requires OpenAI credentials
- **Edge cases or potential failure types:**
  - Standard model/API failures
- **Sub-workflow reference:** None

#### Reporting Agent Tool
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agentTool`; specialist reporting tool.
- **Configuration choices:**
  - Input via `$fromAI('report_request', ...)`
  - System message defines executive summary, findings, risk assessment, recommendations, and audit trail documentation
- **Key expressions or variables used:**
  - `{{ $fromAI('report_request', 'The report request with investigation findings, compliance data, and reporting requirements') }}`
- **Input and output connections:**
  - Receives model input from `Reporting Model`
  - Exposed to `Governance Agent`
- **Version-specific requirements:**
  - Uses type version `3`
- **Edge cases or potential failure types:**
  - Output quality depends heavily on prior tool outputs and source data richness
- **Sub-workflow reference:** None

#### Reporting Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; model for report drafting.
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0.3`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Connected to `Reporting Agent Tool`
- **Version-specific requirements:**
  - Uses type version `1.3`
  - Requires OpenAI credentials
- **Edge cases or potential failure types:**
  - Standard model/API failures
- **Sub-workflow reference:** None

#### Escalation Agent Tool
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agentTool`; specialist escalation assessment tool.
- **Configuration choices:**
  - Input via `$fromAI('escalation_assessment', ...)`
  - System message defines severity, regulatory implications, stakeholder involvement, organizational risk, urgency, and human checkpoints
- **Key expressions or variables used:**
  - `{{ $fromAI('escalation_assessment', 'The case data requiring escalation assessment including risk level, findings, and impact') }}`
- **Input and output connections:**
  - Receives model input from `Escalation Model`
  - Exposed to `Governance Agent`
- **Version-specific requirements:**
  - Uses type version `3`
- **Edge cases or potential failure types:**
  - Escalation severity may drift if prompts are not aligned to formal policy thresholds
- **Sub-workflow reference:** None

#### Escalation Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; model for escalation decisions.
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0.1`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Connected to `Escalation Agent Tool`
- **Version-specific requirements:**
  - Uses type version `1.3`
  - Requires OpenAI credentials
- **Edge cases or potential failure types:**
  - Standard model/API failures
- **Sub-workflow reference:** None

#### Audit Trail Storage Tool
- **Type and technical role:** `n8n-nodes-base.dataTableTool`; AI-callable data-table upsert tool for audit records.
- **Configuration choices:**
  - Operation: `upsert`
  - Mapping mode: explicit field mapping
  - Filter for upsert match: `caseId`
  - Columns populated via `$fromAI(...)`:
    - `action`
    - `caseId`
    - `findings`
    - `agentName`
    - `eventType`
    - `riskLevel`
    - `timestamp`
  - Tool description states it stores and retrieves audit history
- **Key expressions or variables used:**
  - Multiple `$fromAI(...)` expressions for each mapped column
  - Upsert filter: `{{ $fromAI('caseId', 'The unique case identifier to match for upsert') }}`
- **Input and output connections:**
  - Exposed to `Governance Agent` as an AI tool
- **Version-specific requirements:**
  - Uses type version `1.1`
  - Requires an existing n8n Data Table selected by ID
- **Edge cases or potential failure types:**
  - Placeholder table ID must be replaced
  - Upsert behavior depends on schema and uniqueness of `caseId`
  - Timestamp formatting is not normalized automatically
  - Tool may overwrite prior data for the same caseId
- **Sub-workflow reference:** None

#### Policy Database API Tool
- **Type and technical role:** `n8n-nodes-base.httpRequestTool`; AI-callable HTTP tool for policy lookup.
- **Configuration choices:**
  - URL taken from AI via `$fromAI('api_endpoint', ..., 'https://api.example.com/policies')`
  - No authentication or custom method/body configured
  - Tool description: retrieve policy definitions, regulatory requirements, compliance standards
- **Key expressions or variables used:**
  - `{{ $fromAI('api_endpoint', 'The API endpoint to query for policy data', 'string', 'https://api.example.com/policies') }}`
- **Input and output connections:**
  - Exposed to `Governance Agent` as an AI tool
- **Version-specific requirements:**
  - Uses type version `4.4`
- **Edge cases or potential failure types:**
  - Placeholder endpoint must be replaced
  - If the AI chooses an invalid URL, request fails
  - Missing auth headers for real policy systems
  - Network errors, SSL issues, timeouts, non-JSON responses
- **Sub-workflow reference:** None

#### Slack Notification Tool
- **Type and technical role:** `n8n-nodes-base.slackTool`; AI-callable Slack messaging tool.
- **Configuration choices:**
  - Message text from AI: `$fromAI('notification_message', ...)`
  - Channel selected dynamically by AI via `$fromAI('notification_channel', ..., '<__PLACEHOLDER_VALUE__ethics_alerts_channel__>')`
  - OAuth2 authentication
  - Tool description says it sends governance alerts
- **Key expressions or variables used:**
  - `{{ $fromAI('notification_message', 'The notification message to send') }}`
  - `{{ $fromAI('notification_channel', 'The Slack channel ID or name for notifications', 'string', '<__PLACEHOLDER_VALUE__ethics_alerts_channel__>') }}`
- **Input and output connections:**
  - Exposed to `Governance Agent` as an AI tool
- **Version-specific requirements:**
  - Uses type version `2.4`
  - Requires Slack OAuth2 credentials
- **Edge cases or potential failure types:**
  - Placeholder channel must be replaced or AI must supply a valid one
  - Slack permission issues
  - Bot not invited to channel
  - Invalid channel ID/name resolution
- **Sub-workflow reference:** None

---

## 2.3 Structured Decisioning and Risk Routing

### Overview
This block converts the governance result into operational branches. The routing depends entirely on the parsed `riskLevel` field.

### Nodes Involved
- Risk Level Router

### Node Details

#### Risk Level Router
- **Type and technical role:** `n8n-nodes-base.switch`; conditional branch routing.
- **Configuration choices:**
  - Four named outputs:
    - `critical`
    - `high`
    - `medium`
    - `low`
  - Each compares `{{ $json.riskLevel }}` to the matching string
- **Key expressions or variables used:**
  - `{{ $json.riskLevel }}`
- **Input and output connections:**
  - Input from `Governance Agent`
  - `critical` output goes to `Prepare Critical Case Data`
  - `high` output also goes to `Prepare Critical Case Data`
  - `medium` output goes to `Prepare Standard Case Data`
  - `low` output also goes to `Prepare Standard Case Data`
- **Version-specific requirements:**
  - Uses type version `3.4`
- **Edge cases or potential failure types:**
  - If `riskLevel` is null, misspelled, or outside enum, no branch will match
  - Strict type validation means non-string values can break routing
- **Sub-workflow reference:** None

---

## 2.4 Critical Case Handling

### Overview
This block prepares high-severity cases for persistent storage and immediate oversight notification. It is used for both `critical` and `high` risk branches.

### Nodes Involved
- Prepare Critical Case Data
- Store Critical Cases
- Alert Oversight Team

### Node Details

#### Prepare Critical Case Data
- **Type and technical role:** `n8n-nodes-base.set`; normalizes output fields for critical-case persistence and alerting.
- **Configuration choices:**
  - Creates:
    - `caseId` = `{{ $json.body.caseId || $now.toISO() }}`
    - `timestamp` = `{{ $now.toISO() }}`
    - `riskLevel` = `{{ $json.riskLevel }}`
    - `decision` = `{{ $json.decision }}`
    - `reasoning` = `{{ $json.reasoning }}`
    - `escalationPriority` = `{{ $json.escalationPriority }}`
    - `investigationRequired` = `{{ $json.investigationRequired }}`
    - `nextActions` = `{{ JSON.stringify($json.nextActions) }}`
    - `rawDisclosure` = `{{ JSON.stringify($json.body) }}`
- **Key expressions or variables used:**
  - `{{ $json.body.caseId || $now.toISO() }}`
  - `{{ $json.nextActions }}`
  - `{{ JSON.stringify($json.body) }}`
- **Input and output connections:**
  - Input from `Risk Level Router`
  - Output to `Store Critical Cases`
- **Version-specific requirements:**
  - Uses type version `3.4`
- **Edge cases or potential failure types:**
  - Important data dependency issue: after the Governance Agent and output parser, `body` may not still be present unless carried through by n8n item merging semantics; if absent, fallback timestamp becomes `caseId`
  - `nextActions` is declared as type `array` but assigned `JSON.stringify($json.nextActions)`, which produces a string; this may create type inconsistencies
  - `rawDisclosure` is declared as object but is assigned a stringified JSON string
- **Sub-workflow reference:** None

#### Store Critical Cases
- **Type and technical role:** `n8n-nodes-base.dataTable`; stores critical/high-risk case records.
- **Configuration choices:**
  - Mapping mode: `autoMapInputData`
  - Writes to selected Data Table ID placeholder for critical cases
- **Key expressions or variables used:** None directly
- **Input and output connections:**
  - Input from `Prepare Critical Case Data`
  - Output to `Alert Oversight Team`
- **Version-specific requirements:**
  - Uses type version `1.1`
  - Requires an existing n8n Data Table
- **Edge cases or potential failure types:**
  - Placeholder table ID must be replaced
  - Table schema must be compatible with incoming mapped fields
  - Type mismatches from previous Set node may cause insert errors
- **Sub-workflow reference:** None

#### Alert Oversight Team
- **Type and technical role:** `n8n-nodes-base.slack`; sends a formatted critical alert to Slack.
- **Configuration choices:**
  - OAuth2 authentication
  - Sends to fixed placeholder channel ID for critical alerts
  - Message includes:
    - Case ID
    - Risk level
    - Decision
    - Reasoning
    - Escalation priority
    - Investigation flag
    - Next actions
- **Key expressions or variables used:**
  - `{{ $json.caseId }}`
  - `{{ $json.riskLevel }}`
  - `{{ $json.decision }}`
  - `{{ $json.reasoning }}`
  - `{{ $json.escalationPriority }}`
  - `{{ $json.investigationRequired }}`
  - `{{ $json.nextActions.join('\n- ') }}`
- **Input and output connections:**
  - Input from `Store Critical Cases`
  - Output to `Merge All Outcomes`
- **Version-specific requirements:**
  - Uses type version `2.4`
  - Requires Slack OAuth2 credentials
- **Edge cases or potential failure types:**
  - Placeholder channel ID must be replaced
  - `nextActions.join(...)` will fail if `nextActions` is a string rather than an array; this is likely because the prior Set node stringifies it
  - Slack bot permissions or channel membership problems
- **Sub-workflow reference:** None

---

## 2.5 Standard Case Handling

### Overview
This block handles medium and low-risk cases. It prepares a normalized record and stores it without triggering the dedicated critical alert.

### Nodes Involved
- Prepare Standard Case Data
- Store Standard Cases

### Node Details

#### Prepare Standard Case Data
- **Type and technical role:** `n8n-nodes-base.set`; normalizes medium/low-risk case records.
- **Configuration choices:**
  - Creates:
    - `caseId` = `{{ $json.body.caseId || $now.toISO() }}`
    - `timestamp` = `{{ $now.toISO() }}`
    - `riskLevel` = `{{ $json.riskLevel }}`
    - `decision` = `{{ $json.decision }}`
    - `reasoning` = `{{ $json.reasoning }}`
    - `nextActions` = `{{ $json.nextActions }}`
    - `rawDisclosure` = `{{ JSON.stringify($json.body) }}`
- **Key expressions or variables used:**
  - `{{ $json.body.caseId || $now.toISO() }}`
  - `{{ $json.nextActions }}`
  - `{{ JSON.stringify($json.body) }}`
- **Input and output connections:**
  - Input from `Risk Level Router`
  - Output to `Store Standard Cases`
- **Version-specific requirements:**
  - Uses type version `3.4`
- **Edge cases or potential failure types:**
  - Same possible loss of original `body` context as in the critical branch
  - `rawDisclosure` is typed as object but assigned a JSON string
- **Sub-workflow reference:** None

#### Store Standard Cases
- **Type and technical role:** `n8n-nodes-base.dataTable`; stores medium/low-risk cases.
- **Configuration choices:**
  - Mapping mode: `autoMapInputData`
  - Writes to selected Data Table ID placeholder for standard cases
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input from `Prepare Standard Case Data`
  - Output to `Merge All Outcomes`
- **Version-specific requirements:**
  - Uses type version `1.1`
  - Requires an existing n8n Data Table
- **Edge cases or potential failure types:**
  - Placeholder table ID must be replaced
  - Incoming data type mismatch can break inserts
- **Sub-workflow reference:** None

---

## 2.6 Merge and Final API Response

### Overview
This block rejoins the critical and standard branches and sends a final JSON response back to the original webhook caller.

### Nodes Involved
- Merge All Outcomes
- Send Response

### Node Details

#### Merge All Outcomes
- **Type and technical role:** `n8n-nodes-base.merge`; merges branch outputs before response.
- **Configuration choices:**
  - Default merge behavior; no explicit mode configured
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input 1 from `Alert Oversight Team`
  - Input 2 from `Store Standard Cases`
  - Output to `Send Response`
- **Version-specific requirements:**
  - Uses type version `3.2`
- **Edge cases or potential failure types:**
  - Merge behavior depends on execution timing and default mode
  - If a branch does not emit due to routing, developers should confirm merge semantics still allow the active branch through
- **Sub-workflow reference:** None

#### Send Response
- **Type and technical role:** `n8n-nodes-base.respondToWebhook`; returns final API response.
- **Configuration choices:**
  - Responds with JSON
  - Response body:
    - `status: 'processed'`
    - `caseId: $json.caseId`
    - `riskLevel: $json.riskLevel`
    - `decision: $json.decision`
    - `message: 'Ethics disclosure processed successfully'`
- **Key expressions or variables used:**
  - `{{ { status: 'processed', caseId: $json.caseId, riskLevel: $json.riskLevel, decision: $json.decision, message: 'Ethics disclosure processed successfully' } }}`
- **Input and output connections:**
  - Input from `Merge All Outcomes`
- **Version-specific requirements:**
  - Uses type version `1.1`
  - Requires upstream webhook to be configured with `responseNode`
- **Edge cases or potential failure types:**
  - If merge does not emit an item, webhook caller gets no response
  - If prior nodes fail, this node is never reached
- **Sub-workflow reference:** None

---

## 2.7 Documentation and In-Canvas Notes

### Overview
These are non-executing sticky notes embedded in the canvas. They provide setup, architectural, and operational guidance.

### Nodes Involved
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4
- Sticky Note5
- Sticky Note6

### Node Details

#### Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`; visual documentation only
- **Configuration choices:** Contains prerequisites, use cases, customization, and benefits
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note1
- Same technical behavior as above; contains setup steps.

#### Sticky Note2
- Same technical behavior as above; contains end-to-end explanation.

#### Sticky Note3
- Same technical behavior as above; documents risk routing.

#### Sticky Note4
- Same technical behavior as above; documents reporting and escalation stage.

#### Sticky Note5
- Same technical behavior as above; documents monitoring and investigation stage.

#### Sticky Note6
- Same technical behavior as above; documents storage, alerting, and response stage.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Ethics Disclosure Webhook | Webhook | Receives incoming ethics disclosures via POST |  | Governance Agent | ## How It Works<br>This workflow automates ethics disclosure intake, investigation, risk routing, and escalation for compliance officers, legal teams, and ethics oversight boards. Disclosures arrive via webhook and are processed by a central Governance Agent with persistent memory, supported by four specialised AI sub-agents: Ethics Monitoring Agent (flags policy breaches), Investigation Agent (conducts structured inquiry), Reporting Agent (generates case summaries), and Escalation Agent (determines escalation need). Shared tools include Audit Trail Storage, Policy Database API, and Slack Notification Tool. A Governance Output Parser structures results for a Risk Level Router, which splits cases into critical and standard tracks. Critical cases trigger Slack alerts to the oversight team; all cases are stored and merged before a final response is dispatched. This eliminates manual triage, ensures consistent policy application, and maintains a complete audit trail for regulatory accountability.<br>## Monitor & Investigate<br>**What** — Ethics Monitoring Agent flags violations; Investigation Agent conducts structured inquiry.<br>**Why** — Separates detection from investigation to ensure thorough, unbiased case handling. |
| Governance Agent | LangChain Agent | Central orchestration agent for case analysis and tool use | Ethics Disclosure Webhook; Governance Model; Governance Memory; Ethics Monitoring Agent Tool; Investigation Agent Tool; Reporting Agent Tool; Escalation Agent Tool; Audit Trail Storage Tool; Policy Database API Tool; Slack Notification Tool; Governance Output Parser | Risk Level Router | ## Report & Escalate<br>**What** — Reporting Agent drafts case summary; Escalation Agent determines escalation path.<br>**Why** — Produces consistent, policy-aligned outputs before risk routing begins. |
| Governance Model | OpenAI Chat Model | Primary LLM for Governance Agent |  | Governance Agent | ## Monitor & Investigate<br>**What** — Ethics Monitoring Agent flags violations; Investigation Agent conducts structured inquiry.<br>**Why** — Separates detection from investigation to ensure thorough, unbiased case handling. |
| Governance Memory | Buffer Window Memory | Supplies conversational memory to Governance Agent |  | Governance Agent | ## Monitor & Investigate<br>**What** — Ethics Monitoring Agent flags violations; Investigation Agent conducts structured inquiry.<br>**Why** — Separates detection from investigation to ensure thorough, unbiased case handling. |
| Governance Output Parser | Structured Output Parser | Forces agent output into schema with risk and decision fields |  | Governance Agent | ## Report & Escalate<br>**What** — Reporting Agent drafts case summary; Escalation Agent determines escalation path.<br>**Why** — Produces consistent, policy-aligned outputs before risk routing begins.<br>## Risk-Level Routing<br>**What** — Governance Output Parser structures results; Risk Level Router splits into critical and standard tracks.<br>**Why** — Ensures proportionate response — critical cases receive immediate oversight attention. |
| Ethics Monitoring Agent Tool | Agent Tool | Specialized disclosure and policy risk evaluator | Ethics Monitoring Model | Governance Agent | ## Monitor & Investigate<br>**What** — Ethics Monitoring Agent flags violations; Investigation Agent conducts structured inquiry.<br>**Why** — Separates detection from investigation to ensure thorough, unbiased case handling. |
| Ethics Monitoring Model | OpenAI Chat Model | LLM backing Ethics Monitoring tool |  | Ethics Monitoring Agent Tool | ## Monitor & Investigate<br>**What** — Ethics Monitoring Agent flags violations; Investigation Agent conducts structured inquiry.<br>**Why** — Separates detection from investigation to ensure thorough, unbiased case handling. |
| Investigation Agent Tool | Agent Tool | Specialized ethics investigation assistant | Investigation Model | Governance Agent | ## Monitor & Investigate<br>**What** — Ethics Monitoring Agent flags violations; Investigation Agent conducts structured inquiry.<br>**Why** — Separates detection from investigation to ensure thorough, unbiased case handling. |
| Investigation Model | OpenAI Chat Model | LLM backing Investigation tool |  | Investigation Agent Tool | ## Monitor & Investigate<br>**What** — Ethics Monitoring Agent flags violations; Investigation Agent conducts structured inquiry.<br>**Why** — Separates detection from investigation to ensure thorough, unbiased case handling. |
| Reporting Agent Tool | Agent Tool | Specialized compliance report generator | Reporting Model | Governance Agent | ## Report & Escalate<br>**What** — Reporting Agent drafts case summary; Escalation Agent determines escalation path.<br>**Why** — Produces consistent, policy-aligned outputs before risk routing begins. |
| Reporting Model | OpenAI Chat Model | LLM backing Reporting tool |  | Reporting Agent Tool | ## Report & Escalate<br>**What** — Reporting Agent drafts case summary; Escalation Agent determines escalation path.<br>**Why** — Produces consistent, policy-aligned outputs before risk routing begins. |
| Escalation Agent Tool | Agent Tool | Specialized escalation assessment assistant | Escalation Model | Governance Agent | ## Report & Escalate<br>**What** — Reporting Agent drafts case summary; Escalation Agent determines escalation path.<br>**Why** — Produces consistent, policy-aligned outputs before risk routing begins. |
| Escalation Model | OpenAI Chat Model | LLM backing Escalation tool |  | Escalation Agent Tool | ## Report & Escalate<br>**What** — Reporting Agent drafts case summary; Escalation Agent determines escalation path.<br>**Why** — Produces consistent, policy-aligned outputs before risk routing begins. |
| Audit Trail Storage Tool | Data Table Tool | AI-callable audit trail upsert tool |  | Governance Agent | ## Report & Escalate<br>**What** — Reporting Agent drafts case summary; Escalation Agent determines escalation path.<br>**Why** — Produces consistent, policy-aligned outputs before risk routing begins. |
| Policy Database API Tool | HTTP Request Tool | AI-callable policy lookup endpoint |  | Governance Agent | ## Report & Escalate<br>**What** — Reporting Agent drafts case summary; Escalation Agent determines escalation path.<br>**Why** — Produces consistent, policy-aligned outputs before risk routing begins. |
| Slack Notification Tool | Slack Tool | AI-callable generic Slack notifier |  | Governance Agent | ## Report & Escalate<br>**What** — Reporting Agent drafts case summary; Escalation Agent determines escalation path.<br>**Why** — Produces consistent, policy-aligned outputs before risk routing begins. |
| Risk Level Router | Switch | Routes structured cases by risk level | Governance Agent | Prepare Critical Case Data; Prepare Standard Case Data | ## Risk-Level Routing<br>**What** — Governance Output Parser structures results; Risk Level Router splits into critical and standard tracks.<br>**Why** — Ensures proportionate response — critical cases receive immediate oversight attention. |
| Prepare Critical Case Data | Set | Normalizes critical/high case fields | Risk Level Router | Store Critical Cases | ## Risk-Level Routing<br>**What** — Governance Output Parser structures results; Risk Level Router splits into critical and standard tracks.<br>**Why** — Ensures proportionate response — critical cases receive immediate oversight attention.<br>## Store, Alert & Respond<br>**What** — Critical cases alert Slack and store separately; all cases merge before final response is sent.<br>**Why** — Closes the loop with oversight teams and maintains a unified, auditable case record. |
| Store Critical Cases | Data Table | Persists critical/high case records | Prepare Critical Case Data | Alert Oversight Team | ## Store, Alert & Respond<br>**What** — Critical cases alert Slack and store separately; all cases merge before final response is sent.<br>**Why** — Closes the loop with oversight teams and maintains a unified, auditable case record. |
| Alert Oversight Team | Slack | Sends critical case alert to oversight channel | Store Critical Cases | Merge All Outcomes | ## Store, Alert & Respond<br>**What** — Critical cases alert Slack and store separately; all cases merge before final response is sent.<br>**Why** — Closes the loop with oversight teams and maintains a unified, auditable case record. |
| Prepare Standard Case Data | Set | Normalizes medium/low case fields | Risk Level Router | Store Standard Cases | ## Risk-Level Routing<br>**What** — Governance Output Parser structures results; Risk Level Router splits into critical and standard tracks.<br>**Why** — Ensures proportionate response — critical cases receive immediate oversight attention.<br>## Store, Alert & Respond<br>**What** — Critical cases alert Slack and store separately; all cases merge before final response is sent.<br>**Why** — Closes the loop with oversight teams and maintains a unified, auditable case record. |
| Store Standard Cases | Data Table | Persists medium/low case records | Prepare Standard Case Data | Merge All Outcomes | ## Store, Alert & Respond<br>**What** — Critical cases alert Slack and store separately; all cases merge before final response is sent.<br>**Why** — Closes the loop with oversight teams and maintains a unified, auditable case record. |
| Merge All Outcomes | Merge | Rejoins case branches before responding | Alert Oversight Team; Store Standard Cases | Send Response | ## Store, Alert & Respond<br>**What** — Critical cases alert Slack and store separately; all cases merge before final response is sent.<br>**Why** — Closes the loop with oversight teams and maintains a unified, auditable case record. |
| Send Response | Respond to Webhook | Returns final JSON API response | Merge All Outcomes |  | ## Store, Alert & Respond<br>**What** — Critical cases alert Slack and store separately; all cases merge before final response is sent.<br>**Why** — Closes the loop with oversight teams and maintains a unified, auditable case record. |
| Sticky Note | Sticky Note | Canvas documentation for prerequisites and benefits |  |  | ## Prerequisites<br>- Slack workspace and bot token<br>- Ethics policy database or API endpoint<br>- Database or Google Sheets for case and audit storage<br>## Use Cases<br>- Automated triage and escalation of employee ethics disclosures in regulated industries<br>## Customisation<br>- Adjust Risk Level Router thresholds to match organisational severity definitions<br>## Benefits<br>- Eliminates manual disclosure triage — processes cases consistently at scale |
| Sticky Note1 | Sticky Note | Canvas documentation for setup |  |  | ## Setup Steps<br>1. Configure webhook URL in **Ethics Disclosure Webhook** with secure authentication.<br>2. Set AI model credentials (OpenAI/Anthropic) in all agent and model nodes.<br>3. Connect Slack credentials and oversight channel.<br>4. Configure **Policy Database API** with your organisation's ethics policy endpoint or dataset.<br>5. Connect database/Google Sheets credentials<br>6. Test with sample disclosure payloads across both risk tracks before activating. |
| Sticky Note2 | Sticky Note | Canvas documentation for workflow behavior |  |  | ## How It Works<br>This workflow automates ethics disclosure intake, investigation, risk routing, and escalation for compliance officers, legal teams, and ethics oversight boards. Disclosures arrive via webhook and are processed by a central Governance Agent with persistent memory, supported by four specialised AI sub-agents: Ethics Monitoring Agent (flags policy breaches), Investigation Agent (conducts structured inquiry), Reporting Agent (generates case summaries), and Escalation Agent (determines escalation need). Shared tools include Audit Trail Storage, Policy Database API, and Slack Notification Tool. A Governance Output Parser structures results for a Risk Level Router, which splits cases into critical and standard tracks. Critical cases trigger Slack alerts to the oversight team; all cases are stored and merged before a final response is dispatched. This eliminates manual triage, ensures consistent policy application, and maintains a complete audit trail for regulatory accountability. |
| Sticky Note3 | Sticky Note | Canvas documentation for routing logic |  |  | ## Risk-Level Routing<br>**What** — Governance Output Parser structures results; Risk Level Router splits into critical and standard tracks.<br>**Why** — Ensures proportionate response — critical cases receive immediate oversight attention. |
| Sticky Note4 | Sticky Note | Canvas documentation for reporting and escalation |  |  | ## Report & Escalate<br>**What** — Reporting Agent drafts case summary; Escalation Agent determines escalation path.<br>**Why** — Produces consistent, policy-aligned outputs before risk routing begins. |
| Sticky Note5 | Sticky Note | Canvas documentation for monitoring and investigation |  |  | ## Monitor & Investigate<br>**What** — Ethics Monitoring Agent flags violations; Investigation Agent conducts structured inquiry.<br>**Why** — Separates detection from investigation to ensure thorough, unbiased case handling. |
| Sticky Note6 | Sticky Note | Canvas documentation for storage and closure |  |  | ## Store, Alert & Respond<br>**What** — Critical cases alert Slack and store separately; all cases merge before final response is sent.<br>**Why** — Closes the loop with oversight teams and maintains a unified, auditable case record. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Intelligent ethics disclosure governance and escalation agent`.

2. **Add a Webhook node**
   - Type: `Webhook`
   - Name: `Ethics Disclosure Webhook`
   - Method: `POST`
   - Path: `ethics-disclosure`
   - Response Mode: `Using Respond to Webhook Node` / `responseNode`
   - Keep the payload in JSON format.
   - If exposing publicly, add authentication at the workflow or reverse-proxy level because this node itself has no auth configured.

3. **Add the main AI agent**
   - Type: `AI Agent` / `LangChain Agent`
   - Name: `Governance Agent`
   - Set input text to:
     - `{{ $json.body }}`
   - Enable structured output parser.
   - Set system message to a governance orchestration prompt covering:
     - ethics monitoring
     - investigation
     - reporting
     - escalation
     - explainability
     - human authority checkpoints

4. **Add the main OpenAI model**
   - Type: `OpenAI Chat Model`
   - Name: `Governance Model`
   - Model: `gpt-4o`
   - Temperature: `0.2`
   - Connect it to the `Governance Agent` on the `ai_languageModel` port.
   - Configure OpenAI credentials.

5. **Add memory for the agent**
   - Type: `Buffer Window Memory`
   - Name: `Governance Memory`
   - Default configuration is acceptable.
   - Connect it to the `Governance Agent` on the `ai_memory` port.

6. **Add a structured output parser**
   - Type: `Structured Output Parser`
   - Name: `Governance Output Parser`
   - Use manual schema with these fields:
     - `decision`: string
     - `riskLevel`: string enum of `low`, `medium`, `high`, `critical`
     - `investigationRequired`: boolean
     - `escalationPriority`: string
     - `reasoning`: string
     - `nextActions`: array of strings
   - Connect it to the `Governance Agent` on the `ai_outputParser` port.

7. **Add the Ethics Monitoring specialized tool**
   - Type: `Agent Tool`
   - Name: `Ethics Monitoring Agent Tool`
   - Text/input:
     - `{{ $fromAI('disclosure_data', 'The disclosure data to evaluate including conflicts, declarations, and policy signals') }}`
   - System prompt should instruct the tool to analyze:
     - financial conflicts
     - relationship disclosures
     - policy adherence
     - regulatory compliance
   - Add a tool description explaining it returns risk scores, compliance flags, and policy assessments.

8. **Add the model for Ethics Monitoring**
   - Type: `OpenAI Chat Model`
   - Name: `Ethics Monitoring Model`
   - Model: `gpt-4o`
   - Temperature: `0.1`
   - Connect it to `Ethics Monitoring Agent Tool`.
   - Connect `Ethics Monitoring Agent Tool` to `Governance Agent` via `ai_tool`.

9. **Add the Investigation specialized tool**
   - Type: `Agent Tool`
   - Name: `Investigation Agent Tool`
   - Text/input:
     - `{{ $fromAI('investigation_request', 'The investigation request with case details, flagged issues, and scope') }}`
   - System prompt should cover:
     - evidence collection
     - stakeholder interviews
     - timeline reconstruction
     - impact assessment
     - mitigation recommendations
   - Add a tool description indicating full audit-trail-style investigation support.

10. **Add the model for Investigation**
    - Type: `OpenAI Chat Model`
    - Name: `Investigation Model`
    - Model: `gpt-4o`
    - Temperature: `0.2`
    - Connect it to `Investigation Agent Tool`.
    - Connect `Investigation Agent Tool` to `Governance Agent` via `ai_tool`.

11. **Add the Reporting specialized tool**
    - Type: `Agent Tool`
    - Name: `Reporting Agent Tool`
    - Text/input:
      - `{{ $fromAI('report_request', 'The report request with investigation findings, compliance data, and reporting requirements') }}`
    - System prompt should require:
      - executive summary
      - findings and evidence
      - risk assessment
      - recommendations
      - audit trail documentation
    - Add a tool description indicating report generation for stakeholder and compliance review.

12. **Add the model for Reporting**
    - Type: `OpenAI Chat Model`
    - Name: `Reporting Model`
    - Model: `gpt-4o`
    - Temperature: `0.3`
    - Connect it to `Reporting Agent Tool`.
    - Connect `Reporting Agent Tool` to `Governance Agent` via `ai_tool`.

13. **Add the Escalation specialized tool**
    - Type: `Agent Tool`
    - Name: `Escalation Agent Tool`
    - Text/input:
      - `{{ $fromAI('escalation_assessment', 'The case data requiring escalation assessment including risk level, findings, and impact') }}`
    - System prompt should instruct it to determine escalation levels based on:
      - severity
      - regulatory implications
      - stakeholder involvement
      - organizational risk
      - urgency
      - required human checkpoints
    - Add a tool description for escalation routing and oversight recommendation.

14. **Add the model for Escalation**
    - Type: `OpenAI Chat Model`
    - Name: `Escalation Model`
    - Model: `gpt-4o`
    - Temperature: `0.1`
    - Connect it to `Escalation Agent Tool`.
    - Connect `Escalation Agent Tool` to `Governance Agent` via `ai_tool`.

15. **Add the audit trail data tool**
    - Type: `Data Table Tool`
    - Name: `Audit Trail Storage Tool`
    - Operation: `Upsert`
    - Select or create a Data Table for ethics audit events.
    - Configure explicit columns:
      - `action` = `{{ $fromAI('action', 'Action taken or event description') }}`
      - `caseId` = `{{ $fromAI('caseId', 'Unique case identifier') }}`
      - `findings` = `{{ $fromAI('findings', 'Investigation findings or event details') }}`
      - `agentName` = `{{ $fromAI('agentName', 'Name of the agent performing the action') }}`
      - `eventType` = `{{ $fromAI('eventType', 'Type of event (e.g., disclosure, investigation, escalation)') }}`
      - `riskLevel` = `{{ $fromAI('riskLevel', 'Risk level assessment (low, medium, high, critical)') }}`
      - `timestamp` = `{{ $fromAI('timestamp', 'Event timestamp') }}`
    - Add an upsert filter on `caseId`.
    - Connect it to `Governance Agent` via `ai_tool`.

16. **Add the policy lookup tool**
    - Type: `HTTP Request Tool`
    - Name: `Policy Database API Tool`
    - URL:
      - `{{ $fromAI('api_endpoint', 'The API endpoint to query for policy data', 'string', 'https://api.example.com/policies') }}`
    - Replace the default URL with your real policy API endpoint if possible.
    - Add authentication headers if your API requires them.
    - Connect it to `Governance Agent` via `ai_tool`.

17. **Add the AI-callable Slack tool**
    - Type: `Slack Tool`
    - Name: `Slack Notification Tool`
    - Authentication: OAuth2
    - Text:
      - `{{ $fromAI('notification_message', 'The notification message to send') }}`
    - Channel:
      - `{{ $fromAI('notification_channel', 'The Slack channel ID or name for notifications', 'string', '<your_default_ethics_alerts_channel>') }}`
    - Configure Slack OAuth2 credentials.
    - Connect it to `Governance Agent` via `ai_tool`.

18. **Connect the webhook to the Governance Agent**
    - Main output of `Ethics Disclosure Webhook` → main input of `Governance Agent`.

19. **Add a Switch node for risk routing**
    - Type: `Switch`
    - Name: `Risk Level Router`
    - Create 4 outputs with renamed keys:
      - `critical` where `{{ $json.riskLevel }}` equals `critical`
      - `high` where `{{ $json.riskLevel }}` equals `high`
      - `medium` where `{{ $json.riskLevel }}` equals `medium`
      - `low` where `{{ $json.riskLevel }}` equals `low`
    - Use strict string comparison.
    - Connect `Governance Agent` main output to `Risk Level Router`.

20. **Add Set node for critical/high cases**
    - Type: `Set`
    - Name: `Prepare Critical Case Data`
    - Add fields:
      - `caseId` = `{{ $json.body.caseId || $now.toISO() }}`
      - `timestamp` = `{{ $now.toISO() }}`
      - `riskLevel` = `{{ $json.riskLevel }}`
      - `decision` = `{{ $json.decision }}`
      - `reasoning` = `{{ $json.reasoning }}`
      - `escalationPriority` = `{{ $json.escalationPriority }}`
      - `investigationRequired` = `{{ $json.investigationRequired }}`
      - `nextActions` = `{{ JSON.stringify($json.nextActions) }}`
      - `rawDisclosure` = `{{ JSON.stringify($json.body) }}`
    - Connect both `critical` and `high` outputs from `Risk Level Router` to this node.
    - Recommended improvement: store `nextActions` as an actual array, not a string, if you want to use `.join()` later.

21. **Add Data Table node for critical/high storage**
    - Type: `Data Table`
    - Name: `Store Critical Cases`
    - Mapping mode: auto-map input
    - Select or create a dedicated critical-cases table
    - Ensure the table schema supports the fields from the prior Set node.
    - Connect `Prepare Critical Case Data` → `Store Critical Cases`.

22. **Add Slack node for oversight escalation**
    - Type: `Slack`
    - Name: `Alert Oversight Team`
    - Authentication: OAuth2
    - Select a fixed channel for critical cases
    - Message template:
      - Case ID
      - Risk Level
      - Decision
      - Reasoning
      - Escalation Priority
      - Investigation Required
      - Next Actions
    - Equivalent expression:
      - `🚨 CRITICAL ETHICS ALERT ... {{ $json.nextActions.join('\n- ') }}`
    - Connect `Store Critical Cases` → `Alert Oversight Team`.
    - Recommended fix: if `nextActions` was stringified upstream, convert it back to array or do not stringify it in the Set node.

23. **Add Set node for standard cases**
    - Type: `Set`
    - Name: `Prepare Standard Case Data`
    - Add fields:
      - `caseId` = `{{ $json.body.caseId || $now.toISO() }}`
      - `timestamp` = `{{ $now.toISO() }}`
      - `riskLevel` = `{{ $json.riskLevel }}`
      - `decision` = `{{ $json.decision }}`
      - `reasoning` = `{{ $json.reasoning }}`
      - `nextActions` = `{{ $json.nextActions }}`
      - `rawDisclosure` = `{{ JSON.stringify($json.body) }}`
    - Connect both `medium` and `low` outputs from `Risk Level Router` to this node.

24. **Add Data Table node for standard-case storage**
    - Type: `Data Table`
    - Name: `Store Standard Cases`
    - Mapping mode: auto-map input
    - Select or create a standard-cases table
    - Connect `Prepare Standard Case Data` → `Store Standard Cases`.

25. **Add a Merge node**
    - Type: `Merge`
    - Name: `Merge All Outcomes`
    - Leave default settings unless your n8n version requires a specific merge mode.
    - Connect:
      - `Alert Oversight Team` → input 1
      - `Store Standard Cases` → input 2

26. **Add a Respond to Webhook node**
    - Type: `Respond to Webhook`
    - Name: `Send Response`
    - Respond with JSON
    - Response body:
      - `status: 'processed'`
      - `caseId: {{ $json.caseId }}`
      - `riskLevel: {{ $json.riskLevel }}`
      - `decision: {{ $json.decision }}`
      - `message: 'Ethics disclosure processed successfully'`
    - Connect `Merge All Outcomes` → `Send Response`.

27. **Configure credentials**
    - **OpenAI**
      - Required on:
        - Governance Model
        - Ethics Monitoring Model
        - Investigation Model
        - Reporting Model
        - Escalation Model
    - **Slack OAuth2**
      - Required on:
        - Slack Notification Tool
        - Alert Oversight Team
    - **Data Tables**
      - Create or select:
        - ethics audit trail table
        - critical cases table
        - standard cases table
    - **Policy API**
      - Add URL, auth headers, and any required request configuration

28. **Replace all placeholders**
    - Replace:
      - audit trail table placeholder
      - critical cases table placeholder
      - standard cases table placeholder
      - default Slack channels
      - example policy API URL

29. **Test with sample payloads**
    - Send POST requests with JSON bodies that include at least:
      - disclosure text/details
      - optional `caseId`
    - Test all four risk outputs, especially:
      - `critical`
      - `high`
      - `medium`
      - `low`

30. **Validate edge behavior before production**
    - Confirm the Governance Agent always emits schema-valid JSON
    - Confirm `riskLevel` is always one of the four allowed values
    - Confirm merge behavior works when only one branch is active
    - Confirm Slack alert formatting works with your `nextActions` data type
    - Confirm original disclosure data is still available when later nodes reference `$json.body`

### Sub-workflow setup
This workflow does **not** call any sub-workflows and is **not** a sub-workflow itself. All AI specializations are implemented as tool nodes within the same workflow.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Prerequisites: Slack workspace and bot token; ethics policy database or API endpoint; database or Google Sheets for case and audit storage | Canvas note |
| Use case: automated triage and escalation of employee ethics disclosures in regulated industries | Canvas note |
| Customisation: adjust Risk Level Router thresholds to match organisational severity definitions | Canvas note |
| Benefit: eliminates manual disclosure triage and processes cases consistently at scale | Canvas note |
| Setup steps: configure webhook URL securely, set AI credentials in all agent/model nodes, connect Slack credentials and oversight channel, configure Policy Database API, connect database or Google Sheets credentials, and test with sample disclosure payloads before activation | Canvas note |
| Workflow behavior summary: intake via webhook, AI governance orchestration with specialized agents and shared tools, structured parsing, risk routing, storage, critical alerting, merge, and final response | Canvas note |

## Additional implementation observations
| Note Content | Context or Link |
|---|---|
| The workflow title provided by the user differs from the internal workflow name in JSON. The JSON name is `Intelligent ethics disclosure governance and escalation agent`. | Naming consistency |
| There is a likely type mismatch in `Prepare Critical Case Data`: `nextActions` is typed as array but populated with `JSON.stringify(...)`. This may break the Slack alert node’s `.join()` call. | Important fix before production |
| Both Set nodes reference `$json.body`, but depending on agent output shaping, the original webhook body may not persist automatically. Consider preserving the original disclosure explicitly before AI processing if reliable traceability is required. | Data integrity |
| The Policy Database API Tool currently uses an example endpoint `https://api.example.com/policies`, which must be replaced for real use. | External integration |
| The workflow is currently inactive (`active: false`). | Deployment status |
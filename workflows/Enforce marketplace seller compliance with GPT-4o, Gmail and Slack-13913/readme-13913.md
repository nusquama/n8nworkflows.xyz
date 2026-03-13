Enforce marketplace seller compliance with GPT-4o, Gmail and Slack

https://n8nworkflows.xyz/workflows/enforce-marketplace-seller-compliance-with-gpt-4o--gmail-and-slack-13913


# Enforce marketplace seller compliance with GPT-4o, Gmail and Slack

# 1. Workflow Overview

This workflow automates seller compliance enforcement for a marketplace. It receives seller case data through a webhook, uses a central AI governance agent with several specialized AI sub-agents to assess policy violations and appeals, produces a structured enforcement decision, logs the result for auditability, notifies internal stakeholders, sends external seller communications, and updates seller compliance records.

Typical use cases include:
- Reviewing seller behavior for policy violations
- Evaluating appeals against prior enforcement
- Determining appropriate actions such as warnings or suspensions
- Maintaining an auditable compliance history
- Notifying compliance teams and sellers consistently

## 1.1 Input Reception

The workflow starts from an HTTP webhook that accepts seller compliance payloads via `POST`.

## 1.2 Governance Orchestration and AI Decisioning

A central Governance Agent receives the seller payload and coordinates multiple AI tools:
- Policy Monitoring Agent
- Appeals Review Agent
- Enforcement Decision Agent

It also uses:
- A memory node for conversational context
- A structured output parser to enforce a strict JSON schema
- OpenAI chat models for all agent/tool reasoning

## 1.3 Policy and Violation Analysis

The Policy Monitoring Agent analyzes seller behavior and compliance signals. It has access to:
- A calculator tool
- A custom code tool that computes a severity score and classifies risk

This block supports objective scoring and evidence-based policy assessment.

## 1.4 Appeals and Enforcement Decision

Two specialized agents support the orchestration:
- Appeals Review Agent evaluates appeal fairness and new evidence
- Enforcement Decision Agent recommends a proportional action

These are exposed as tools to the Governance Agent.

## 1.5 Routing and Audit Logging

Once the Governance Agent emits structured output, a Switch node routes all supported enforcement actions to a common audit preparation step. The resulting record is written to a data table for traceability.

## 1.6 Notifications and Seller Record Update

After the audit entry is created:
- Slack notifies the internal compliance team
- A second Switch routes seller-facing email notifications
- The workflow updates seller compliance records in a data table

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception

### Overview
This block receives seller compliance case data from an external system. It is the single entry point of the workflow and passes the inbound payload to the central governance logic.

### Nodes Involved
- Receive Seller Data

### Node Details

#### Receive Seller Data
- **Type and role:** `n8n-nodes-base.webhook`; entry-point webhook for incoming seller compliance events.
- **Configuration choices:**
  - HTTP method: `POST`
  - Path: `governance/seller-compliance`
  - No additional options configured
- **Key expressions or variables used:**
  - Downstream nodes expect the request body under `{{$json.body}}`
- **Input and output connections:**
  - No input node; this is the workflow trigger
  - Outputs to: `Governance Agent`
- **Version-specific requirements:**
  - Uses webhook node type version `2.1`
- **Edge cases / failures:**
  - Payload may not be in the format expected by downstream AI nodes
  - Missing or malformed `body` may cause poor model performance or parser failures
  - No authentication is configured in the node JSON; this is a security gap if exposed publicly
- **Sub-workflow reference:** None

---

## 2.2 Governance Orchestration and Structured AI Output

### Overview
This block is the core reasoning engine. The Governance Agent consumes the seller payload, can call specialized sub-agents as tools, uses memory for context, and is forced to emit a validated structured enforcement object.

### Nodes Involved
- Governance Agent
- Governance Model
- Governance Memory
- Structured Enforcement Output
- Output Parser Model

### Node Details

#### Governance Agent
- **Type and role:** `@n8n/n8n-nodes-langchain.agent`; orchestrating AI agent that coordinates all compliance decisioning.
- **Configuration choices:**
  - Prompt source: defined directly in the node
  - Input text: `={{ $json.body }}`
  - System message defines governance responsibilities:
    - coordinate sub-agents
    - ensure fairness and non-discrimination
    - maintain auditability
    - escalate high-stakes cases
    - balance seller rights and marketplace integrity
  - Output parser enabled
- **Key expressions or variables used:**
  - `{{$json.body}}` as main case input
- **Input and output connections:**
  - Main input from: `Receive Seller Data`
  - AI language model input from: `Governance Model`
  - AI memory input from: `Governance Memory`
  - AI tools from:
    - `Policy Monitoring Agent`
    - `Appeals Review Agent`
    - `Enforcement Decision Agent`
  - AI output parser from: `Structured Enforcement Output`
  - Main output to: `Route Enforcement Action`
- **Version-specific requirements:**
  - Uses type version `3.1`
  - Requires n8n LangChain-compatible AI agent support
- **Edge cases / failures:**
  - If incoming body is unstructured, tool invocation may be inconsistent
  - Model may omit required schema fields without parser correction
  - High token usage possible if payload is large
  - Any failure in connected AI tool/model/parser nodes will break orchestration
- **Sub-workflow reference:** None

#### Governance Model
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; primary LLM for the Governance Agent.
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0.2`
  - No built-in tools enabled
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Outputs via AI language model connection to: `Governance Agent`
- **Version-specific requirements:**
  - Uses type version `1.3`
  - Requires valid OpenAI credentials
- **Edge cases / failures:**
  - OpenAI auth or quota failure
  - Model availability changes
  - Large prompts may hit token or cost constraints
- **Sub-workflow reference:** None

#### Governance Memory
- **Type and role:** `@n8n/n8n-nodes-langchain.memoryBufferWindow`; conversational memory for the governance agent.
- **Configuration choices:**
  - Defaults used; no custom parameters set
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Outputs via AI memory connection to: `Governance Agent`
- **Version-specific requirements:**
  - Type version `1.3`
- **Edge cases / failures:**
  - Since no explicit memory settings are shown, actual retention behavior depends on defaults
  - Memory may not provide durable long-term state across separate executions
- **Sub-workflow reference:** None

#### Structured Enforcement Output
- **Type and role:** `@n8n/n8n-nodes-langchain.outputParserStructured`; validates and normalizes the Governance Agent response into a strict schema.
- **Configuration choices:**
  - Manual JSON schema
  - Auto-fix enabled
  - Required output fields:
    - `seller_id`
    - `violation_type`
    - `severity`
    - `enforcement_action`
    - `reasoning`
    - `requires_human_review`
  - Optional fields include `evidence` and `timestamp`
  - Allowed `severity` values:
    - `low`
    - `medium`
    - `high`
    - `critical`
  - Allowed `enforcement_action` values:
    - `warning`
    - `review`
    - `suspension`
    - `termination`
    - `appeal_approved`
    - `appeal_denied`
- **Key expressions or variables used:** Manual schema only
- **Input and output connections:**
  - AI language model input from: `Output Parser Model`
  - Outputs via AI output parser connection to: `Governance Agent`
- **Version-specific requirements:**
  - Type version `1.3`
- **Edge cases / failures:**
  - Parser may still fail if agent output is too malformed
  - `autoFix` can repair minor formatting issues but not missing reasoning context
  - If agent returns unsupported enum values, routing later will fail or return no branch
- **Sub-workflow reference:** None

#### Output Parser Model
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; lightweight LLM used by the structured output parser.
- **Configuration choices:**
  - Model: `gpt-4o-mini`
  - Temperature: `0`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Outputs via AI language model connection to: `Structured Enforcement Output`
- **Version-specific requirements:**
  - Type version `1.3`
  - Requires OpenAI credentials
- **Edge cases / failures:**
  - Auth/quota failures
  - Parser model may still fail on extremely inconsistent agent output
- **Sub-workflow reference:** None

---

## 2.3 Policy and Violation Analysis

### Overview
This block provides policy assessment support to the Governance Agent. The Policy Monitoring Agent can analyze seller metrics and invoke quantitative tools to compute severity and validate compliance signals.

### Nodes Involved
- Policy Monitoring Agent
- Policy Monitoring Model
- Compliance Calculator
- Violation Severity Scorer

### Node Details

#### Policy Monitoring Agent
- **Type and role:** `@n8n/n8n-nodes-langchain.agentTool`; specialized AI tool agent focused on policy validation and compliance analysis.
- **Configuration choices:**
  - Input text uses AI-supplied variable:
    - `={{ $fromAI('seller_data', 'Seller behavior and compliance data to validate') }}`
  - Tool description explains that it validates behavior, detects violations, and assesses severity
  - System message emphasizes objective, evidence-based policy assessment
- **Key expressions or variables used:**
  - `$fromAI('seller_data', ...)`
- **Input and output connections:**
  - AI language model input from: `Policy Monitoring Model`
  - AI tool inputs from:
    - `Compliance Calculator`
    - `Violation Severity Scorer`
  - Outputs via AI tool connection to: `Governance Agent`
- **Version-specific requirements:**
  - Type version `3`
- **Edge cases / failures:**
  - If the Governance Agent does not pass sufficient seller data, analysis quality drops
  - Tool arguments extracted by AI may be incomplete or type-inconsistent
- **Sub-workflow reference:** None

#### Policy Monitoring Model
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; model used by the Policy Monitoring Agent.
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0.2`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Outputs via AI language model connection to: `Policy Monitoring Agent`
- **Version-specific requirements:**
  - Type version `1.3`
  - Requires OpenAI credentials
- **Edge cases / failures:**
  - Standard OpenAI auth, latency, quota, or model availability issues
- **Sub-workflow reference:** None

#### Compliance Calculator
- **Type and role:** `@n8n/n8n-nodes-langchain.toolCalculator`; arithmetic helper available to the Policy Monitoring Agent.
- **Configuration choices:**
  - No custom configuration
- **Key expressions or variables used:** None directly
- **Input and output connections:**
  - Outputs via AI tool connection to: `Policy Monitoring Agent`
- **Version-specific requirements:**
  - Type version `1`
- **Edge cases / failures:**
  - Limited to calculation-oriented use cases
  - Wrong numeric extraction by AI can still produce misleading results
- **Sub-workflow reference:** None

#### Violation Severity Scorer
- **Type and role:** `@n8n/n8n-nodes-langchain.toolCode`; custom JavaScript scoring tool for compliance severity.
- **Configuration choices:**
  - Reads AI-provided numeric inputs:
    - `return_rate`
    - `complaint_count`
    - `violation_history`
    - `avg_response_time`
  - Defaults:
    - `return_rate = 0`
    - `complaint_count = 0`
    - `violation_history = 0`
    - `avg_response_time = 24`
  - Computes a score from 0–100 using weighted factors:
    - returns up to 40 points
    - complaints up to 30 points
    - violation history up to 20 points
    - response delay up to 10 points
  - Classifies severity:
    - `<25`: low
    - `<50`: medium
    - `<75`: high
    - otherwise critical
- **Key expressions or variables used:**
  - `$fromAI('return_rate', ..., 'number', 0)`
  - `$fromAI('complaint_count', ..., 'number', 0)`
  - `$fromAI('violation_history', ..., 'number', 0)`
  - `$fromAI('avg_response_time', ..., 'number', 24)`
- **Input and output connections:**
  - Outputs via AI tool connection to: `Policy Monitoring Agent`
- **Version-specific requirements:**
  - Type version `1.3`
  - Requires support for LangChain code tools
- **Edge cases / failures:**
  - AI may pass bad numeric values or unexpected units
  - Negative values are not explicitly guarded against
  - The scoring model is heuristic and may not match policy/legal expectations without calibration
- **Sub-workflow reference:** None

---

## 2.4 Appeals and Enforcement Decision

### Overview
This block separates appeal evaluation from enforcement recommendation. Both agents are callable by the Governance Agent, helping it preserve fairness and proportionality in final decisions.

### Nodes Involved
- Appeals Review Agent
- Appeals Review Model
- Enforcement Decision Agent
- Enforcement Decision Model

### Node Details

#### Appeals Review Agent
- **Type and role:** `@n8n/n8n-nodes-langchain.agentTool`; specialized AI tool for appeal evaluation.
- **Configuration choices:**
  - Input text:
    - `={{ $fromAI('appeal_data', 'Appeal submission with seller arguments and evidence') }}`
  - System instructions emphasize fairness, fresh review, procedural checks, and benefit of doubt when evidence is ambiguous
  - Tool description defines appeal approval/denial recommendation behavior
- **Key expressions or variables used:**
  - `$fromAI('appeal_data', ...)`
- **Input and output connections:**
  - AI language model input from: `Appeals Review Model`
  - Outputs via AI tool connection to: `Governance Agent`
- **Version-specific requirements:**
  - Type version `3`
- **Edge cases / failures:**
  - If no appeal data exists, the agent may still be called with insufficient context
  - Ambiguous evidence may lead to inconsistent recommendations
- **Sub-workflow reference:** None

#### Appeals Review Model
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; model backing the Appeals Review Agent.
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0.3`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Outputs via AI language model connection to: `Appeals Review Agent`
- **Version-specific requirements:**
  - Type version `1.3`
  - Requires OpenAI credentials
- **Edge cases / failures:**
  - Standard OpenAI credential, quota, timeout, or latency issues
- **Sub-workflow reference:** None

#### Enforcement Decision Agent
- **Type and role:** `@n8n/n8n-nodes-langchain.agentTool`; specialized AI tool to choose the enforcement action.
- **Configuration choices:**
  - Input text:
    - `={{ $fromAI('violation_assessment', 'Violation assessment and seller context for enforcement decision') }}`
  - System message defines progressive discipline, mitigating factors, consistency, and human escalation requirements
  - Tool description covers warning/review/suspension/termination decisions
- **Key expressions or variables used:**
  - `$fromAI('violation_assessment', ...)`
- **Input and output connections:**
  - AI language model input from: `Enforcement Decision Model`
  - Outputs via AI tool connection to: `Governance Agent`
- **Version-specific requirements:**
  - Type version `3`
- **Edge cases / failures:**
  - If seller history is absent, progressive discipline may be weakly grounded
  - Human-review escalation is advisory only; there is no enforced pause in this workflow
- **Sub-workflow reference:** None

#### Enforcement Decision Model
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; model used by the Enforcement Decision Agent.
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0.2`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Outputs via AI language model connection to: `Enforcement Decision Agent`
- **Version-specific requirements:**
  - Type version `1.3`
  - Requires OpenAI credentials
- **Edge cases / failures:**
  - Standard OpenAI issues: auth, quota, token size, timeout
- **Sub-workflow reference:** None

---

## 2.5 Enforcement Routing and Audit Logging

### Overview
This block normalizes all supported enforcement outcomes into one common post-decision path. It prepares a compact audit payload and writes it to a data table before notifications occur.

### Nodes Involved
- Route Enforcement Action
- Prepare Audit Log
- Enforcement Audit Trail

### Node Details

#### Route Enforcement Action
- **Type and role:** `n8n-nodes-base.switch`; routes by `enforcement_action`.
- **Configuration choices:**
  - Six explicit string-equality branches:
    - `warning`
    - `review`
    - `suspension`
    - `termination`
    - `appeal_approved`
    - `appeal_denied`
  - All branches currently converge to the same next node: `Prepare Audit Log`
- **Key expressions or variables used:**
  - `={{ $json.enforcement_action }}`
- **Input and output connections:**
  - Input from: `Governance Agent`
  - Outputs to: `Prepare Audit Log` on all six branches
- **Version-specific requirements:**
  - Type version `3.4`
- **Edge cases / failures:**
  - Unsupported action values will not match any branch
  - Because all outputs converge, the switch currently acts more like validation than differentiated routing
  - `review` and `termination` have no dedicated seller notification branch later
- **Sub-workflow reference:** None

#### Prepare Audit Log
- **Type and role:** `n8n-nodes-base.set`; formats a structured audit record.
- **Configuration choices:**
  - Assigns:
    - `seller_id`
    - `violation_type`
    - `severity`
    - `enforcement_action`
    - `reasoning`
    - `evidence` as JSON string
    - `requires_human_review`
    - `timestamp` using current execution time
    - `workflow_execution_id` from execution metadata
- **Key expressions or variables used:**
  - `={{ $json.seller_id }}`
  - `={{ $json.violation_type }}`
  - `={{ $json.severity }}`
  - `={{ $json.enforcement_action }}`
  - `={{ $json.reasoning }}`
  - `={{ JSON.stringify($json.evidence) }}`
  - `={{ $json.requires_human_review }}`
  - `={{ $now.toISO() }}`
  - `={{ $execution.id }}`
- **Input and output connections:**
  - Input from: `Route Enforcement Action`
  - Output to: `Enforcement Audit Trail`
- **Version-specific requirements:**
  - Type version `3.4`
- **Edge cases / failures:**
  - If `evidence` is undefined, `JSON.stringify(undefined)` may not behave as intended for storage
  - Any missing required fields from prior parsing reduce audit quality
- **Sub-workflow reference:** None

#### Enforcement Audit Trail
- **Type and role:** `n8n-nodes-base.dataTable`; persists audit records.
- **Configuration choices:**
  - Data table operation appears to be insert-like default behavior
  - Column mapping mode: auto-map input data
  - Data table ID is placeholder-based and must be configured
- **Key expressions or variables used:** None directly
- **Input and output connections:**
  - Input from: `Prepare Audit Log`
  - Outputs to:
    - `Notify Compliance Team`
    - `Route Notifications`
- **Version-specific requirements:**
  - Type version `1.1`
  - Requires n8n Data Table feature availability
- **Edge cases / failures:**
  - Placeholder table ID must be replaced
  - Schema mismatch between input fields and table columns can fail writes
  - If this node fails, notifications and seller record updates will not proceed
- **Sub-workflow reference:** None

---

## 2.6 Notifications and Seller Record Management

### Overview
This block handles internal Slack alerts, seller-facing Gmail notifications, and compliance-record persistence. Different enforcement actions produce different external communications, then all notification paths update a seller record store.

### Nodes Involved
- Notify Compliance Team
- Route Notifications
- Send Warning Email
- Send Suspension Notice
- Send Appeal Decision
- Prepare Seller Record
- Seller Compliance Records

### Node Details

#### Notify Compliance Team
- **Type and role:** `n8n-nodes-base.slack`; internal alert to compliance staff.
- **Configuration choices:**
  - OAuth2 authentication
  - Sends to a configured channel ID placeholder
  - Message includes seller ID, action, violation type, severity, reasoning, human review flag, and execution ID
- **Key expressions or variables used:**
  - `{{ $json.seller_id }}`
  - `{{ $json.enforcement_action }}`
  - `{{ $json.violation_type }}`
  - `{{ $json.severity }}`
  - `{{ $json.reasoning }}`
  - `{{ $json.requires_human_review ? 'Yes ⚠️' : 'No' }}`
  - `{{ $json.workflow_execution_id }}`
- **Input and output connections:**
  - Input from: `Enforcement Audit Trail`
  - No downstream output connected
- **Version-specific requirements:**
  - Type version `2.4`
  - Requires Slack OAuth2 credentials and a valid channel selection
- **Edge cases / failures:**
  - Placeholder channel must be replaced
  - Missing Slack scope or channel access causes posting failures
  - Long reasoning text may reduce readability in Slack
- **Sub-workflow reference:** None

#### Route Notifications
- **Type and role:** `n8n-nodes-base.switch`; routes seller-facing notifications by action type.
- **Configuration choices:**
  - Four branches:
    - `warning`
    - `suspension`
    - `appeal_approved`
    - `appeal_denied`
  - Appeals approved and denied both route to the same Gmail node
- **Key expressions or variables used:**
  - `={{ $json.enforcement_action }}`
- **Input and output connections:**
  - Input from: `Enforcement Audit Trail`
  - Outputs to:
    - `Send Warning Email`
    - `Send Suspension Notice`
    - `Send Appeal Decision`
    - `Send Appeal Decision`
- **Version-specific requirements:**
  - Type version `3.4`
- **Edge cases / failures:**
  - `review` and `termination` actions do not trigger seller-facing email here
  - Unsupported action values result in no seller notification
- **Sub-workflow reference:** None

#### Send Warning Email
- **Type and role:** `n8n-nodes-base.gmail`; sends policy warning to seller.
- **Configuration choices:**
  - Recipient is a placeholder email value
  - Subject: `Policy Violation Warning - Action Required`
  - Body contains violation type, severity, reasoning, evidence list, correction request, and appeal guidance
- **Key expressions or variables used:**
  - `{{ $json.violation_type }}`
  - `{{ $json.severity }}`
  - `{{ $json.reasoning }}`
  - `{{ $json.evidence ? $json.evidence.join('\n') : 'See attached details' }}`
- **Input and output connections:**
  - Input from: `Route Notifications`
  - Output to: `Prepare Seller Record`
- **Version-specific requirements:**
  - Type version `2.2`
  - Requires Gmail OAuth2 credentials
- **Edge cases / failures:**
  - `sendTo` is currently a placeholder and must be dynamically sourced or replaced
  - If `evidence` is a string from previous storage transformation rather than array, `.join()` will fail
  - Gmail sending limits and OAuth expiry can interrupt delivery
- **Sub-workflow reference:** None

#### Send Suspension Notice
- **Type and role:** `n8n-nodes-base.gmail`; sends seller suspension notice.
- **Configuration choices:**
  - Recipient is a placeholder email value
  - Subject: `Account Suspension Notice - Immediate Action Required`
  - Body explains immediate suspension, appeal process, and required evidence
- **Key expressions or variables used:**
  - `{{ $json.violation_type }}`
  - `{{ $json.severity }}`
  - `{{ $json.reasoning }}`
  - `{{ $json.evidence ? $json.evidence.join('\n') : 'See attached details' }}`
- **Input and output connections:**
  - Input from: `Route Notifications`
  - Output to: `Prepare Seller Record`
- **Version-specific requirements:**
  - Type version `2.2`
  - Requires Gmail OAuth2 credentials
- **Edge cases / failures:**
  - Same email placeholder and evidence-type issues as warning email
  - Delivery failures could leave internal records updated only partially depending on execution mode
- **Sub-workflow reference:** None

#### Send Appeal Decision
- **Type and role:** `n8n-nodes-base.gmail`; sends either appeal approval or denial.
- **Configuration choices:**
  - Recipient: placeholder expression form
  - Subject dynamically becomes `Approved` or `Denied`
  - Body uses a conditional expression:
    - approved: account restrictions lifted
    - denied: original action remains
- **Key expressions or variables used:**
  - `{{ $json.enforcement_action === 'appeal_approved' ? ... : ... }}`
- **Input and output connections:**
  - Input from: `Route Notifications`
  - Output to: `Prepare Seller Record`
- **Version-specific requirements:**
  - Type version `2.2`
  - Requires Gmail OAuth2 credentials
- **Edge cases / failures:**
  - Placeholder recipient must be replaced
  - If `enforcement_action` is unexpected, the "denied" branch becomes the default message
- **Sub-workflow reference:** None

#### Prepare Seller Record
- **Type and role:** `n8n-nodes-base.set`; maps final case outcome into a seller-status record.
- **Configuration choices:**
  - Assigns:
    - `seller_id`
    - `current_status`
    - `last_violation_type`
    - `last_violation_severity`
    - `last_action_date`
    - `total_violations`
    - `requires_monitoring`
  - `total_violations` is computed as `($json.total_violations || 0) + 1`
  - `requires_monitoring` is true for `high` or `critical`
- **Key expressions or variables used:**
  - `={{ $json.seller_id }}`
  - `={{ $json.enforcement_action }}`
  - `={{ $json.violation_type }}`
  - `={{ $json.severity }}`
  - `={{ $now.toISO() }}`
  - `={{ ($json.total_violations || 0) + 1 }}`
  - `={{ $json.severity === 'high' || $json.severity === 'critical' }}`
- **Input and output connections:**
  - Inputs from:
    - `Send Warning Email`
    - `Send Suspension Notice`
    - `Send Appeal Decision`
  - Output to: `Seller Compliance Records`
- **Version-specific requirements:**
  - Type version `3.4`
- **Edge cases / failures:**
  - `total_violations` does not actually read existing table state in this workflow; it increments only if upstream payload already contains the field
  - `review` and `termination` actions never reach this node because they are not routed through seller notifications
- **Sub-workflow reference:** None

#### Seller Compliance Records
- **Type and role:** `n8n-nodes-base.dataTable`; upserts seller compliance profile/status.
- **Configuration choices:**
  - Operation: `upsert`
  - Filter condition:
    - `seller_id = {{$json.seller_id}}`
  - Column mapping: auto-map input data
  - Data table ID is placeholder-based and must be set
- **Key expressions or variables used:**
  - Filter on `seller_id`
- **Input and output connections:**
  - Input from: `Prepare Seller Record`
  - No downstream nodes
- **Version-specific requirements:**
  - Type version `1.1`
  - Requires n8n Data Table support
- **Edge cases / failures:**
  - Placeholder table ID must be replaced
  - Upsert behavior depends on the target table schema and unique key expectations
  - Because only some action types reach this node, records may not be updated for `review` or `termination`
- **Sub-workflow reference:** None

---

## 2.7 Documentation Sticky Notes

### Overview
These nodes are non-executable annotations embedded in the canvas. They document setup, architecture, and operating intent, and should be preserved for maintainability.

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
- **Type and role:** `n8n-nodes-base.stickyNote`; documents prerequisites, use cases, customization, and benefits.
- **Configuration choices:** Contains text only.
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases / failures:** None
- **Sub-workflow reference:** None

#### Sticky Note1
- **Type and role:** `n8n-nodes-base.stickyNote`; documents setup steps.
- **Configuration choices:** Contains text only.
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases / failures:** None
- **Sub-workflow reference:** None

#### Sticky Note2
- **Type and role:** `n8n-nodes-base.stickyNote`; documents the end-to-end workflow behavior.
- **Configuration choices:** Contains text only.
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases / failures:** None
- **Sub-workflow reference:** None

#### Sticky Note3
- **Type and role:** `n8n-nodes-base.stickyNote`; describes audit and routing section.
- **Configuration choices:** Contains text only.
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases / failures:** None
- **Sub-workflow reference:** None

#### Sticky Note4
- **Type and role:** `n8n-nodes-base.stickyNote`; describes appeals and enforcement section.
- **Configuration choices:** Contains text only.
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases / failures:** None
- **Sub-workflow reference:** None

#### Sticky Note5
- **Type and role:** `n8n-nodes-base.stickyNote`; describes policy and violation analysis.
- **Configuration choices:** Contains text only.
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases / failures:** None
- **Sub-workflow reference:** None

#### Sticky Note6
- **Type and role:** `n8n-nodes-base.stickyNote`; describes notifications and record updates.
- **Configuration choices:** Contains text only.
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases / failures:** None
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Receive Seller Data | webhook | Receives incoming seller compliance payload via HTTP POST |  | Governance Agent | ## How It Works<br>This workflow automates marketplace seller compliance monitoring and enforcement for platform trust, legal, and operations teams. It receives seller data via webhook, routes it through a central Governance Agent backed by persistent memory, and fans out to four specialised AI sub-agents: Policy Monitoring (with compliance scoring and violation severity calculation), Appeals Review, and Enforcement Decision. A Structured Enforcement Output parser standardises results before routing to enforcement actions. The workflow then prepares an audit log, writes to an Enforcement Audit Trail, and triggers multi-channel notifications — Gmail appeal decisions, warning emails, Slack alerts to the compliance team, and suspension notices. Finally, seller records are updated in a Seller Compliance Records store. This eliminates manual case reviews, ensures consistent policy application, and creates a full auditable enforcement trail at scale.<br>## Policy & Violation Analysis<br>**What** — Policy Monitoring Agent scores compliance; Violation Severity Scorer calculates risk level.<br>**Why** — Quantifies risk objectively, removing human bias from initial triage. |
| Governance Agent | langchain agent | Central orchestrator for compliance assessment and final decision | Receive Seller Data; Governance Model; Governance Memory; Policy Monitoring Agent; Appeals Review Agent; Enforcement Decision Agent; Structured Enforcement Output | Route Enforcement Action | ## Policy & Violation Analysis<br>**What** — Policy Monitoring Agent scores compliance; Violation Severity Scorer calculates risk level.<br>**Why** — Quantifies risk objectively, removing human bias from initial triage. |
| Governance Model | OpenAI Chat Model | LLM powering the Governance Agent |  | Governance Agent | ## Policy & Violation Analysis<br>**What** — Policy Monitoring Agent scores compliance; Violation Severity Scorer calculates risk level.<br>**Why** — Quantifies risk objectively, removing human bias from initial triage. |
| Governance Memory | memoryBufferWindow | Provides short-term memory to the Governance Agent |  | Governance Agent | ## Policy & Violation Analysis<br>**What** — Policy Monitoring Agent scores compliance; Violation Severity Scorer calculates risk level.<br>**Why** — Quantifies risk objectively, removing human bias from initial triage. |
| Structured Enforcement Output | structured output parser | Enforces structured final decision schema | Output Parser Model | Governance Agent | ## Appeals & Enforcement Decision<br>**What** — Appeals Review Agent and Enforcement Decision Agent evaluate case in parallel.<br>**Why** — Separates appeals logic from enforcement to ensure fair, independent assessment. |
| Policy Monitoring Agent | langchain agent tool | Specialized tool for policy validation and compliance assessment | Policy Monitoring Model; Compliance Calculator; Violation Severity Scorer | Governance Agent | ## Policy & Violation Analysis<br>**What** — Policy Monitoring Agent scores compliance; Violation Severity Scorer calculates risk level.<br>**Why** — Quantifies risk objectively, removing human bias from initial triage. |
| Appeals Review Agent | langchain agent tool | Specialized tool for appeal evaluation | Appeals Review Model | Governance Agent | ## Appeals & Enforcement Decision<br>**What** — Appeals Review Agent and Enforcement Decision Agent evaluate case in parallel.<br>**Why** — Separates appeals logic from enforcement to ensure fair, independent assessment. |
| Enforcement Decision Agent | langchain agent tool | Specialized tool for selecting enforcement action | Enforcement Decision Model | Governance Agent | ## Appeals & Enforcement Decision<br>**What** — Appeals Review Agent and Enforcement Decision Agent evaluate case in parallel.<br>**Why** — Separates appeals logic from enforcement to ensure fair, independent assessment. |
| Policy Monitoring Model | OpenAI Chat Model | LLM for policy monitoring analysis |  | Policy Monitoring Agent | ## Policy & Violation Analysis<br>**What** — Policy Monitoring Agent scores compliance; Violation Severity Scorer calculates risk level.<br>**Why** — Quantifies risk objectively, removing human bias from initial triage. |
| Appeals Review Model | OpenAI Chat Model | LLM for appeal review |  | Appeals Review Agent | ## Appeals & Enforcement Decision<br>**What** — Appeals Review Agent and Enforcement Decision Agent evaluate case in parallel.<br>**Why** — Separates appeals logic from enforcement to ensure fair, independent assessment. |
| Enforcement Decision Model | OpenAI Chat Model | LLM for enforcement recommendation |  | Enforcement Decision Agent | ## Appeals & Enforcement Decision<br>**What** — Appeals Review Agent and Enforcement Decision Agent evaluate case in parallel.<br>**Why** — Separates appeals logic from enforcement to ensure fair, independent assessment. |
| Output Parser Model | OpenAI Chat Model | Lightweight model to help structured parsing |  | Structured Enforcement Output | ## Appeals & Enforcement Decision<br>**What** — Appeals Review Agent and Enforcement Decision Agent evaluate case in parallel.<br>**Why** — Separates appeals logic from enforcement to ensure fair, independent assessment. |
| Compliance Calculator | calculator tool | Math helper for compliance analysis |  | Policy Monitoring Agent | ## Policy & Violation Analysis<br>**What** — Policy Monitoring Agent scores compliance; Violation Severity Scorer calculates risk level.<br>**Why** — Quantifies risk objectively, removing human bias from initial triage. |
| Violation Severity Scorer | code tool | Computes numeric severity score and severity label |  | Policy Monitoring Agent | ## Policy & Violation Analysis<br>**What** — Policy Monitoring Agent scores compliance; Violation Severity Scorer calculates risk level.<br>**Why** — Quantifies risk objectively, removing human bias from initial triage. |
| Route Enforcement Action | switch | Branches by final enforcement action | Governance Agent | Prepare Audit Log | ## Audit & Route Actions<br>**What** — Formats audit log, writes to Enforcement Audit Trail, routes enforcement action.<br>**Why** — Guarantees full traceability and triggers the correct downstream response path. |
| Prepare Audit Log | set | Maps final decision into audit record fields | Route Enforcement Action | Enforcement Audit Trail | ## Audit & Route Actions<br>**What** — Formats audit log, writes to Enforcement Audit Trail, routes enforcement action.<br>**Why** — Guarantees full traceability and triggers the correct downstream response path. |
| Enforcement Audit Trail | dataTable | Stores auditable enforcement event records | Prepare Audit Log | Notify Compliance Team; Route Notifications | ## Audit & Route Actions<br>**What** — Formats audit log, writes to Enforcement Audit Trail, routes enforcement action.<br>**Why** — Guarantees full traceability and triggers the correct downstream response path. |
| Notify Compliance Team | slack | Sends internal Slack alert | Enforcement Audit Trail |  | ## Notify & Record<br>**What** — Sends Gmail decisions, warning emails, Slack alerts, suspension notices; updates seller records.<br>**Why** — Closes the loop with all stakeholders and maintains a live compliance database. |
| Send Warning Email | gmail | Sends seller warning notice | Route Notifications | Prepare Seller Record | ## Notify & Record<br>**What** — Sends Gmail decisions, warning emails, Slack alerts, suspension notices; updates seller records.<br>**Why** — Closes the loop with all stakeholders and maintains a live compliance database. |
| Send Suspension Notice | gmail | Sends seller suspension notice | Route Notifications | Prepare Seller Record | ## Notify & Record<br>**What** — Sends Gmail decisions, warning emails, Slack alerts, suspension notices; updates seller records.<br>**Why** — Closes the loop with all stakeholders and maintains a live compliance database. |
| Send Appeal Decision | gmail | Sends appeal approval or denial email | Route Notifications | Prepare Seller Record | ## Notify & Record<br>**What** — Sends Gmail decisions, warning emails, Slack alerts, suspension notices; updates seller records.<br>**Why** — Closes the loop with all stakeholders and maintains a live compliance database. |
| Seller Compliance Records | dataTable | Upserts latest seller compliance state | Prepare Seller Record |  | ## Notify & Record<br>**What** — Sends Gmail decisions, warning emails, Slack alerts, suspension notices; updates seller records.<br>**Why** — Closes the loop with all stakeholders and maintains a live compliance database. |
| Prepare Seller Record | set | Formats seller status record for persistence | Send Warning Email; Send Suspension Notice; Send Appeal Decision | Seller Compliance Records | ## Notify & Record<br>**What** — Sends Gmail decisions, warning emails, Slack alerts, suspension notices; updates seller records.<br>**Why** — Closes the loop with all stakeholders and maintains a live compliance database. |
| Route Notifications | switch | Routes seller-facing notifications by action | Enforcement Audit Trail | Send Warning Email; Send Suspension Notice; Send Appeal Decision | ## Notify & Record<br>**What** — Sends Gmail decisions, warning emails, Slack alerts, suspension notices; updates seller records.<br>**Why** — Closes the loop with all stakeholders and maintains a live compliance database. |
| Sticky Note | stickyNote | Canvas documentation for prerequisites and benefits |  |  | ## Prerequisites<br>- Gmail account with OAuth2 credentials<br>- Slack workspace and bot token<br>- Database or Google Sheets for audit and records storage<br>## Use Cases<br>- Automated suspension and warning issuance for policy-violating marketplace sellers<br>## Customisation<br>- Swap enforcement channels (e.g., replace Gmail with SendGrid)<br>## Benefits<br>- Eliminates manual seller case reviews — scales enforcement without added headcount |
| Sticky Note1 | stickyNote | Canvas documentation for setup steps |  |  | ## Setup Steps<br>1. Configure webhook URL in **Receive Seller Data** node and secure with authentication.<br>2. Set AI model credentials (OpenAI/Anthropic) in all agent and model nodes.<br>3. Add Slack credentials and target channel to **Notify Compliance Team** node.<br>4. Connect database/Google Sheets credentials.<br>5. Activate and test with a sample seller payload. |
| Sticky Note2 | stickyNote | Canvas documentation for workflow behavior |  |  | ## How It Works<br>This workflow automates marketplace seller compliance monitoring and enforcement for platform trust, legal, and operations teams. It receives seller data via webhook, routes it through a central Governance Agent backed by persistent memory, and fans out to four specialised AI sub-agents: Policy Monitoring (with compliance scoring and violation severity calculation), Appeals Review, and Enforcement Decision. A Structured Enforcement Output parser standardises results before routing to enforcement actions. The workflow then prepares an audit log, writes to an Enforcement Audit Trail, and triggers multi-channel notifications — Gmail appeal decisions, warning emails, Slack alerts to the compliance team, and suspension notices. Finally, seller records are updated in a Seller Compliance Records store. This eliminates manual case reviews, ensures consistent policy application, and creates a full auditable enforcement trail at scale. |
| Sticky Note3 | stickyNote | Canvas documentation for audit and routing section |  |  | ## Audit & Route Actions<br>**What** — Formats audit log, writes to Enforcement Audit Trail, routes enforcement action.<br>**Why** — Guarantees full traceability and triggers the correct downstream response path. |
| Sticky Note4 | stickyNote | Canvas documentation for appeals and enforcement section |  |  | ## Appeals & Enforcement Decision<br>**What** — Appeals Review Agent and Enforcement Decision Agent evaluate case in parallel.<br>**Why** — Separates appeals logic from enforcement to ensure fair, independent assessment. |
| Sticky Note5 | stickyNote | Canvas documentation for policy and violation section |  |  | ## Policy & Violation Analysis<br>**What** — Policy Monitoring Agent scores compliance; Violation Severity Scorer calculates risk level.<br>**Why** — Quantifies risk objectively, removing human bias from initial triage. |
| Sticky Note6 | stickyNote | Canvas documentation for notifications and record updates |  |  | ## Notify & Record<br>**What** — Sends Gmail decisions, warning emails, Slack alerts, suspension notices; updates seller records.<br>**Why** — Closes the loop with all stakeholders and maintains a live compliance database. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Intelligent seller governance enforcement and compliance automation`.

2. **Add the webhook trigger**
   - Create a `Webhook` node named `Receive Seller Data`.
   - Set:
     - Method: `POST`
     - Path: `governance/seller-compliance`
   - Recommended hardening:
     - enable authentication or front it with a protected API gateway
     - validate inbound JSON before AI processing

3. **Add the central AI agent**
   - Create an `AI Agent` node named `Governance Agent`.
   - Set prompt mode to define the prompt directly.
   - Set input text to:
     - `{{$json.body}}`
   - Paste a system message describing the governance role:
     - orchestrates policy monitoring, appeals, and enforcement
     - ensures fairness and auditability
     - escalates high-stakes cases
   - Enable structured output parsing.

4. **Add the governance model**
   - Create an `OpenAI Chat Model` node named `Governance Model`.
   - Select model `gpt-4o`.
   - Set temperature to `0.2`.
   - Connect it to the `Governance Agent` using the AI language model connection.
   - Configure OpenAI credentials.

5. **Add memory for the governance agent**
   - Create a `Memory Buffer Window` node named `Governance Memory`.
   - Keep default settings unless you want a larger or smaller memory window.
   - Connect it to the `Governance Agent` via the AI memory connection.

6. **Add the structured output parser**
   - Create a `Structured Output Parser` node named `Structured Enforcement Output`.
   - Set schema mode to manual.
   - Enable auto-fix.
   - Define a schema with these fields:
     - `seller_id` string
     - `violation_type` string
     - `severity` enum: `low`, `medium`, `high`, `critical`
     - `enforcement_action` enum: `warning`, `review`, `suspension`, `termination`, `appeal_approved`, `appeal_denied`
     - `reasoning` string
     - `evidence` array of strings
     - `requires_human_review` boolean
     - `timestamp` string
   - Make required:
     - `seller_id`
     - `violation_type`
     - `severity`
     - `enforcement_action`
     - `reasoning`
     - `requires_human_review`

7. **Add the parser model**
   - Create an `OpenAI Chat Model` node named `Output Parser Model`.
   - Set model to `gpt-4o-mini`.
   - Set temperature to `0`.
   - Connect it to `Structured Enforcement Output` as the parser model.
   - Connect `Structured Enforcement Output` to `Governance Agent` as the AI output parser.

8. **Add the Policy Monitoring tool agent**
   - Create an `AI Agent Tool` node named `Policy Monitoring Agent`.
   - Set text input to:
     - `{{$fromAI('seller_data', 'Seller behavior and compliance data to validate')}}`
   - Add a system message instructing it to:
     - validate seller behavior against policies
     - analyze return rates, complaint ratios, response times, prohibited items, etc.
     - detect fraud/non-compliance
     - assess severity
     - return evidence-based findings
   - Add a tool description summarizing that capability.
   - Connect this node to `Governance Agent` as an AI tool.

9. **Add the Policy Monitoring model**
   - Create an `OpenAI Chat Model` node named `Policy Monitoring Model`.
   - Use `gpt-4o`, temperature `0.2`.
   - Connect it to `Policy Monitoring Agent`.
   - Reuse the same OpenAI credential if desired.

10. **Add the calculator tool**
    - Create a `Calculator Tool` node named `Compliance Calculator`.
    - Leave default configuration.
    - Connect it to `Policy Monitoring Agent` as an AI tool.

11. **Add the severity code tool**
    - Create a `Code Tool` node named `Violation Severity Scorer`.
    - Use JavaScript.
    - Add logic that:
      - reads `return_rate`, `complaint_count`, `violation_history`, and `avg_response_time` from `$fromAI(...)`
      - computes a weighted score up to 100
      - maps score to `low`, `medium`, `high`, or `critical`
      - returns score, severity, and metric breakdown
    - Connect it to `Policy Monitoring Agent` as an AI tool.

12. **Add the Appeals Review tool agent**
    - Create an `AI Agent Tool` node named `Appeals Review Agent`.
    - Set text input to:
      - `{{$fromAI('appeal_data', 'Appeal submission with seller arguments and evidence')}}`
    - Add a system message emphasizing:
      - fresh review
      - fairness
      - mitigation and procedural review
      - benefit of doubt in ambiguous cases
    - Add a tool description about appeal evaluation.
    - Connect it to `Governance Agent` as an AI tool.

13. **Add the Appeals Review model**
    - Create an `OpenAI Chat Model` node named `Appeals Review Model`.
    - Use `gpt-4o`, temperature `0.3`.
    - Connect it to `Appeals Review Agent`.

14. **Add the Enforcement Decision tool agent**
    - Create an `AI Agent Tool` node named `Enforcement Decision Agent`.
    - Set text input to:
      - `{{$fromAI('violation_assessment', 'Violation assessment and seller context for enforcement decision')}}`
    - Add a system message covering:
      - progressive discipline
      - mitigating factors
      - consistency across similar cases
      - human review for suspensions/terminations
    - Add a tool description about deciding warning/review/suspension/termination.
    - Connect it to `Governance Agent` as an AI tool.

15. **Add the Enforcement Decision model**
    - Create an `OpenAI Chat Model` node named `Enforcement Decision Model`.
    - Use `gpt-4o`, temperature `0.2`.
    - Connect it to `Enforcement Decision Agent`.

16. **Connect the trigger to the orchestrator**
    - Connect `Receive Seller Data` main output to `Governance Agent` main input.

17. **Add enforcement-action routing**
    - Create a `Switch` node named `Route Enforcement Action`.
    - Add six conditions on `{{$json.enforcement_action}}`:
      - equals `warning`
      - equals `review`
      - equals `suspension`
      - equals `termination`
      - equals `appeal_approved`
      - equals `appeal_denied`
    - Connect `Governance Agent` to this node.
    - Connect every branch to the same next node, `Prepare Audit Log`.

18. **Add audit-log formatting**
    - Create a `Set` node named `Prepare Audit Log`.
    - Add fields:
      - `seller_id` = `{{$json.seller_id}}`
      - `violation_type` = `{{$json.violation_type}}`
      - `severity` = `{{$json.severity}}`
      - `enforcement_action` = `{{$json.enforcement_action}}`
      - `reasoning` = `{{$json.reasoning}}`
      - `evidence` = `{{JSON.stringify($json.evidence)}}`
      - `requires_human_review` = `{{$json.requires_human_review}}`
      - `timestamp` = `{{$now.toISO()}}`
      - `workflow_execution_id` = `{{$execution.id}}`
    - Connect `Route Enforcement Action` to `Prepare Audit Log`.

19. **Add audit persistence**
    - Create a `Data Table` node named `Enforcement Audit Trail`.
    - Set column mapping to auto-map input fields.
    - Select or create a data table for audit storage.
    - Replace the placeholder with the actual table ID.
    - Connect `Prepare Audit Log` to `Enforcement Audit Trail`.

20. **Add internal Slack notification**
    - Create a `Slack` node named `Notify Compliance Team`.
    - Use OAuth2 authentication.
    - Post to a selected compliance channel.
    - Message should include:
      - seller ID
      - action
      - violation
      - severity
      - reasoning
      - human review flag
      - execution ID
    - Connect `Enforcement Audit Trail` to `Notify Compliance Team`.
    - Configure Slack credentials and ensure the bot can post to the chosen channel.

21. **Add seller notification routing**
    - Create a second `Switch` node named `Route Notifications`.
    - Add four conditions on `{{$json.enforcement_action}}`:
      - `warning`
      - `suspension`
      - `appeal_approved`
      - `appeal_denied`
    - Connect `Enforcement Audit Trail` to `Route Notifications`.

22. **Add warning email**
    - Create a `Gmail` node named `Send Warning Email`.
    - Configure Gmail OAuth2 credentials.
    - Set recipient to the seller email source you want to use.
      - In the JSON it is still a placeholder, so in a real build use something like `{{$json.seller_email}}`.
    - Subject:
      - `Policy Violation Warning - Action Required`
    - Body should include:
      - violation type
      - severity
      - reasoning
      - evidence
      - remediation timeline
      - appeal instructions
    - Connect the `warning` branch from `Route Notifications` to this node.

23. **Add suspension email**
    - Create a `Gmail` node named `Send Suspension Notice`.
    - Use the same Gmail credential.
    - Set seller recipient.
    - Subject:
      - `Account Suspension Notice - Immediate Action Required`
    - Body should explain:
      - violation details
      - immediate suspension
      - appeal deadline
      - evidence/documentation required
    - Connect the `suspension` branch from `Route Notifications` to this node.

24. **Add appeal decision email**
    - Create a `Gmail` node named `Send Appeal Decision`.
    - Use a conditional body expression:
      - if `enforcement_action` is `appeal_approved`, send approval message
      - else send denial message
    - Subject should similarly switch between `Approved` and `Denied`.
    - Connect both `appeal_approved` and `appeal_denied` branches from `Route Notifications` to this node.

25. **Add seller record preparation**
    - Create a `Set` node named `Prepare Seller Record`.
    - Map:
      - `seller_id`
      - `current_status` = `{{$json.enforcement_action}}`
      - `last_violation_type`
      - `last_violation_severity`
      - `last_action_date` = `{{$now.toISO()}}`
      - `total_violations` = `{{($json.total_violations || 0) + 1}}`
      - `requires_monitoring` = `{{$json.severity === 'high' || $json.severity === 'critical'}}`
    - Connect:
      - `Send Warning Email` → `Prepare Seller Record`
      - `Send Suspension Notice` → `Prepare Seller Record`
      - `Send Appeal Decision` → `Prepare Seller Record`

26. **Add seller compliance record storage**
    - Create a `Data Table` node named `Seller Compliance Records`.
    - Set operation to `Upsert`.
    - Add filter:
      - key `seller_id`
      - value `{{$json.seller_id}}`
    - Enable auto-map input fields.
    - Replace the placeholder data table ID with a real table.
    - Connect `Prepare Seller Record` to `Seller Compliance Records`.

27. **Add visual documentation if desired**
    - Create sticky notes for:
      - prerequisites
      - setup steps
      - architecture overview
      - policy analysis
      - appeals/enforcement
      - audit/routing
      - notifications/recording

28. **Configure credentials**
    - **OpenAI**
      - Required for:
        - Governance Model
        - Policy Monitoring Model
        - Appeals Review Model
        - Enforcement Decision Model
        - Output Parser Model
    - **Gmail OAuth2**
      - Required for:
        - Send Warning Email
        - Send Suspension Notice
        - Send Appeal Decision
    - **Slack OAuth2**
      - Required for:
        - Notify Compliance Team
    - **Data Table / storage**
      - Ensure both data tables exist and the n8n instance supports Data Tables

29. **Replace all placeholders**
    - Replace:
      - seller email placeholder
      - compliance Slack channel placeholder
      - enforcement audit trail table ID placeholder
      - seller compliance records table ID placeholder

30. **Test with a sample payload**
    - Use a JSON payload containing at minimum:
      - seller ID
      - violation details
      - compliance metrics
      - appeal data if relevant
      - ideally seller email
    - Confirm the Governance Agent returns valid schema fields.

31. **Validate edge behavior before activation**
    - Test all supported action outputs:
      - `warning`
      - `review`
      - `suspension`
      - `termination`
      - `appeal_approved`
      - `appeal_denied`
    - Important observation:
      - `review` and `termination` currently go to audit logging and Slack, but not to seller-facing email or seller-record update paths.
    - If that is not intentional, add extra branches for those actions.

32. **Activate the workflow**
    - Once credentials, table IDs, and recipient sourcing are correct, activate the workflow and expose the production webhook URL.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Gmail account with OAuth2 credentials required | Prerequisites |
| Slack workspace and bot token required | Prerequisites |
| Database or Google Sheets for audit and records storage | Prerequisites note; actual workflow uses n8n Data Tables |
| Automated suspension and warning issuance for policy-violating marketplace sellers | Use case |
| Swap enforcement channels, for example replacing Gmail with SendGrid | Customisation |
| Eliminates manual seller case reviews and scales enforcement without added headcount | Benefit |
| Configure webhook URL in `Receive Seller Data` and secure it with authentication | Setup guidance |
| Set AI model credentials in all agent and model nodes | Setup guidance |
| Add Slack credentials and target channel to `Notify Compliance Team` | Setup guidance |
| Connect database or storage credentials and tables | Setup guidance |
| Activate and test with a sample seller payload | Setup guidance |
| The workflow uses a central governance agent plus specialized policy, appeals, and enforcement agents | Architecture note |
| The workflow creates an audit trail before notifications | Operational note |
| The workflow currently does not send seller-facing notifications for `review` or `termination` actions | Important implementation note |
| The workflow currently increments `total_violations` only from incoming data, not by reading the existing record first | Data consistency note |
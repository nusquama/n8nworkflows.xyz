Optimize procurement approval workflows with OpenAI, Slack, and Gmail

https://n8nworkflows.xyz/workflows/optimize-procurement-approval-workflows-with-openai--slack--and-gmail-20507


# Optimize procurement approval workflows with OpenAI, Slack, and Gmail

### 1. Workflow Overview

The **Adaptive Procurement Workflow Designer** automates the weekly auditing, performance analysis, and optimization of enterprise procurement approval processes. Its core objective is to balance process speed with compliance controls by detecting approval bottlenecks (such as delayed Finance sign-offs), generating structured optimization proposals using either a built-in mock generator or OpenAI, validating proposals against strict organizational guardrails, waiting for management sign-off via a resume webhook, versioning the configuration, and notifying stakeholders.

The logical blocks comprise:
- **1.1 Input Reception & Data Aggregation:** Triggers weekly, checks for procurement policy changes, and loads historical transaction data, active approval steps, and buying rules.
- **1.2 Performance Analysis & Threshold Evaluation:** Normalizes transactions, calculates per-role bottleneck metrics, scores overall execution performance, and decides whether an optimization cycle is required.
- **1.3 AI Processing & Rule Validation:** Formulates structured context, dispatches prompts to either a demo engine or OpenAI, parses the response, and performs deterministic compliance and safety checks.
- **1.4 Human Approval & Change Gating:** Evaluates safety validation results, creates a formal change request, pauses execution via a webhook waiting for managerial sign-off, and processes the incoming payload.
- **1.5 Versioning, Deployment & Stakeholder Notifications:** Evaluates the manager's decision, creates a preserved version update with rollback snapshots, logs the audit history, and broadcasts notifications via Slack and Gmail.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Data Aggregation
- **Overview:** Initiates the workflow on a weekly schedule, checks for version updates in enterprise buying policies, and populates mock historical procurement transactions alongside current sequential routing rules.
- **Nodes Involved:**
  - `Start Weekly Review`
  - ` Changes Check If Company Policies Changed`
  - `Get Past Purchase Approval History`
  - `Get Current Approval Steps`
  - `Get Company Buying Rules`

- **Node Details:**
  - **Start Weekly Review**
    - *Type and Technical Role:* `n8n-nodes-base.scheduleTrigger` (Trigger Node)
    - *Configuration Choices:* Configured to fire once per week (`weeks` interval).
    - *Key Expressions/Variables:* None.
    - *Connections:* Output connects to ` Changes Check If Company Policies Changed`.
    - *Potential Failures:* None typical; relies on n8n internal scheduler.
  - ** Changes Check If Company Policies Changed**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration Choices:* Evaluates string constants comparing `lastPolicyVersion` ('2.0') to `currentPolicyVersion` ('2.1').
    - *Key Expressions/Variables:* Returns `policyChanged`, `currentPolicyVersion`, and `changeSummary`.
    - *Connections:* Input from `Start Weekly Review`; output to `Get Past Purchase Approval History`.
    - *Potential Failures:* Syntax errors in inline script (handled by n8n runtime).
  - **Get Past Purchase Approval History**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration Choices:* Generates 25 realistic mock procurement transactions with embedded approval steps and role durations.
    - *Key Expressions/Variables:* References upstream policy checks via `$(' Changes Check If Company Policies Changed').first().json`.
    - *Connections:* Input from policy check; output to `Get Current Approval Steps`.
    - *Potential Failures:* Memory allocation issues if transaction volume is artificially scaled up excessively.
  - **Get Current Approval Steps**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration Choices:* Establishes sequential approval rules across amount thresholds (e.g., $0–$1k, $1k–$10k, $10k+) and defines allowed operational roles.
    - *Key Expressions/Variables:* References history data via `$('Get Past Purchase Approval History').first().json`.
    - *Connections:* Input from approval history; output to `Get Company Buying Rules`.
    - *Potential Failures:* Unhandled property access if previous node output format changes.
  - **Get Company Buying Rules**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration Choices:* Sets active governance policies (e.g., high-value controls, restricted categories, audit trail mandates).
    - *Key Expressions/Variables:* References upstream context via `$('Get Current Approval Steps').first().json`.
    - *Connections:* Input from approval steps; output to `Clean and Organize the Data`.
    - *Potential Failures:* Array mapping issues if policy definitions contain invalid structures.

---

#### 2.2 Performance Analysis & Threshold Evaluation
- **Overview:** Cleans and normalizes historical transactions, calculates queue and duration delays, computes per-role bottleneck scores, evaluates overall end-to-end performance metrics, and determines if process optimization is needed.
- **Nodes Involved:**
  - `Clean and Organize the Data`
  - `Find What Is Slowing Approvals`
  - `Score Overall Approval Performance`
  - `Do We Need to Improve the Process?`
  - `No Change Needed — Stop Here`

- **Node Details:**
  - **Clean and Organize the Data**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration Choices:* Normalizes raw transaction payloads into structured events and computes duration aggregates against an 8-hour delay threshold (`480` minutes).
    - *Key Expressions/Variables:* References buying rules data via `$('Get Company Buying Rules').first().json`.
    - *Connections:* Input from buying rules; output to `Find What Is Slowing Approvals`.
    - *Potential Failures:* Division by zero during hour conversions if transaction steps are empty.
  - **Find What Is Slowing Approvals**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration Choices:* Calculates median/average durations, rejection rates, and computes a weighted `bottleneckScore` across roles (`averageDuration*0.40 + delayRate*0.30 + rejectionRate*0.20 + volume*0.10`).
    - *Key Expressions/Variables:* References cleaned data via `$('Clean and Organize the Data').first().json`.
    - *Connections:* Input from data cleaning; output to `Score Overall Approval Performance`.
    - *Potential Failures:* Sorting errors on empty duration arrays.
  - **Score Overall Approval Performance**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration Choices:* Aggregates end-to-end hours, rejection/rework rates, and sets the `optimizationNeeded` boolean flag based on thresholds (e.g., bottleneck score >= 70, average time > 16 hours, or policy changes).
    - *Key Expressions/Variables:* References bottleneck statistics via `$('Find What Is Slowing Approvals').first().json`.
    - *Connections:* Input from bottleneck analysis; output to `Do We Need to Improve the Process?`.
    - *Potential Failures:* Undefined metrics property access.
  - **Do We Need to Improve the Process?**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Conditional Branching)
    - *Configuration Choices:* Evaluates strict boolean condition `={{ $json.optimizationNeeded }}`.
    - *Key Expressions/Variables:* `{{ $json.optimizationNeeded }}`
    - *Connections:* Input from performance scoring. True output goes to `Prepare Information for AI`; False output goes to `No Change Needed — Stop Here`.
    - *Potential Failures:* Type evaluation mismatch if the property is missing from the incoming item.
  - **No Change Needed — Stop Here**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration Choices:* Formats termination payload indicating performance is within acceptable thresholds.
    - *Key Expressions/Variables:* References performance scoring data via `$('Score Overall Approval Performance').first().json`.
    - *Connections:* Input from false branch of `Do We Need to Improve the Process?`. Terminates branch.
    - *Potential Failures:* None.

---

#### 2.3 AI Processing & Rule Validation
- **Overview:** Prepares structured context and prompts for the optimization analyst, routes execution to either a mock recommendation engine or live OpenAI, extracts and parses the JSON response, and validates it against ten deterministic compliance rules.
- **Nodes Involved:**
  - `Prepare Information for AI`
  - `Use Demo AI or Real AI?`
  - `Demo AI Suggestion (No Login Needed)`
  - `Real AI Suggestion (OpenAI)`
  - `Read AI Answer as Structured Data`
  - `Check AI Suggestion Against Company Rules`

- **Node Details:**
  - **Prepare Information for AI**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration Choices:* Compiles enterprise context, active compliance mandates, allowed actions (`ADD`, `REMOVE`, `REORDER`, `PARALLELIZE`, `ROUTE`, `CHANGE_THRESHOLD`), and constructs the system/user prompts.
    - *Key Expressions/Variables:* References performance metrics via `$('Score Overall Approval Performance').first().json`.
    - *Connections:* Input from true branch of `Do We Need to Improve the Process?`; output to `Use Demo AI or Real AI?`.
    - *Potential Failures:* Stringify errors on circular or malformed rule structures.
  - **Use Demo AI or Real AI?**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Conditional Branching)
    - *Configuration Choices:* Evaluates `={{ $json.useMockAi }}` (defaults to `true` for credential-free testing).
    - *Key Expressions/Variables:* `{{ $json.useMockAi }}`
    - *Connections:* Input from AI preparation. True branch (mock mode) connects to `Demo AI Suggestion (No Login Needed)`; False branch connects to `Real AI Suggestion (OpenAI)`.
    - *Potential Failures:* Boolean type resolution failure.
  - **Demo AI Suggestion (No Login Needed)**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration Choices:* Generates a deterministic recommendation JSON proposing to parallelize Manager and Finance while preserving mandatory controls.
    - *Key Expressions/Variables:* References AI prep node.
    - *Connections:* Input from true branch of router; output to `Read AI Answer as Structured Data`.
    - *Potential Failures:* None.
  - **Real AI Suggestion (OpenAI)**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.openAi` (AI Model Integration)
    - *Configuration Choices:* Uses model `gpt-4.1` with a temperature of `0.2` and max tokens `2000`. Configures system and user message parameters.
    - *Key Expressions/Variables:* System prompt: `={{ $json.systemPrompt }}`, User prompt: `={{ $json.userPrompt }}`.
    - *Credentials Required:* `openAiApi`
    - *Connections:* Input from false branch of router; output to `Read AI Answer as Structured Data`.
    - *Potential Failures:* API rate limits, authentication failures, context window length violations, or timeout errors.
  - **Read AI Answer as Structured Data**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration Choices:* Strips markdown code fences (` ```json `), executes robust `JSON.parse()`, and fails closed with validation errors if parsing fails.
    - *Key Expressions/Variables:* Inspects `$input.first().json` and fallback preview properties.
    - *Connections:* Inputs from either `Demo AI Suggestion (No Login Needed)` or `Real AI Suggestion (OpenAI)`; output to `Check AI Suggestion Against Company Rules`.
    - *Potential Failures:* Syntax errors if the LLM output contains invalid JSON structures.
  - **Check AI Suggestion Against Company Rules**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration Choices:* Enforces 10 strict validation rules (checking for disallowed SQL/exec commands, verifying allowed roles/actions, ensuring mandatory roles like Finance/Manager are not removed, enforcing confidence >= 0.80, and checking risk level).
    - *Key Expressions/Variables:* References parsed data from `Read AI Answer as Structured Data`.
    - *Connections:* Input from structured data reader; output to `Is the AI Suggestion Safe to Use?`.
    - *Potential Failures:* Unhandled property lookups if recommendation fields are null.

---

#### 2.4 Human Approval & Change Gating
- **Overview:** Evaluates whether the AI suggestion passed deterministic validation, routes rejected proposals to team notifications, creates a formal change request for approved suggestions, and pauses workflow execution waiting for managerial webhook sign-off.
- **Nodes Involved:**
  - `Is the AI Suggestion Safe to Use?`
  - `Tell Team: AI Suggestion Rejected`
  - `Email Team — AI Suggestion Rejected`
  - `Create Change Request for Approval`
  - `Wait for Manager Approval`
  - `Read Approve / Reject Answer`

- **Node Details:**
  - **Is the AI Suggestion Safe to Use?**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Conditional Branching)
    - *Configuration Choices:* Evaluates `={{ $json.valid }}`.
    - *Key Expressions/Variables:* `{{ $json.valid }}`
    - *Connections:* Input from rule checker. True branch connects to `Create Change Request for Approval`; False branch connects to `Tell Team: AI Suggestion Rejected`.
    - *Potential Failures:* Boolean validation mismatch.
  - **Tell Team: AI Suggestion Rejected**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration Choices:* Formats rejection notification payloads detailing validation errors.
    - *Key Expressions/Variables:* References rule checker data.
    - *Connections:* Input from false branch of safety check; output to `Email Team — AI Suggestion Rejected`.
    - *Potential Failures:* None.
  - **Email Team — AI Suggestion Rejected**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (Email Integration)
    - *Configuration Choices:* Sends plain text email alerts to `info@example.com` regarding validation failures.
    - *KeyExpressions/Variables:* Subject: `={{ $json.notification.subject }}`, Message: `={{ $json.notification.body }}`.
    - *Credentials Required:* `gmailOAuth2`
    - *Connections:* Input from rejection formatter. Terminates branch.
    - *Potential Failures:* SMTP authentication errors, invalid recipient addresses, or rate limits.
  - **Create Change Request for Approval**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration Choices:* Assigns a unique change ID (`CHG-######`), computes the proposed version increment (e.g., `v3` -> `v4`), and flags whether high-risk changes require additional approval.
    - *Key Expressions/Variables:* References validated recommendation data.
    - *Connections:* Input from true branch of safety check; output to `Wait for Manager Approval`.
    - *Potential Failures:* Version parsing errors on non-standard version strings.
  - **Wait for Manager Approval**
    - *Type and Technical Role:* `n8n-nodes-base.wait` (Execution Pause / Webhook Receiver)
    - *Configuration Choices:* Pauses workflow execution awaiting an HTTP POST request to the resume webhook (`adaptive-procurement-human-approval`).
    - *Key Expressions/Variables:* Expects JSON payloads containing `approvalStatus`, `approvedBy`, and optional `additionalApproval`.
    - *Connections:* Input from change request creation; output to `Read Approve / Reject Answer`.
    - *Potential Failures:* Webhook timeouts, unreachable callback URLs, or malformed JSON payloads sent by external managers.
  - **Read Approve / Reject Answer**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration Choices:* Normalizes incoming webhook payloads, validates whether high-risk changes received the required `additionalApproval=APPROVED` flag, and sets the boolean `approved` status.
    - *Key Expressions/Variables:* Parses `$input.first().json.body` or direct webhook properties.
    - *Connections:* Input from wait webhook resumption; output to `Did the Manager Approve?`.
    - *Potential Failures:* Undefined property access if the webhook payload body is missing.

---

#### 2.5 Versioning, Deployment & Stakeholder Notifications
- **Overview:** Evaluates the manager's decision, handles human rejections, creates new process versions with rollback snapshots for approvals, applies configuration updates, writes audit logs, and broadcasts updates via Slack and Gmail.
- **Nodes Involved:**
  - `Did the Manager Approve?`
  - `Tell Team: Change Was Rejected`
  - `Create New Process Version (Keep Old One)`
  - `Apply the New Approval Process`
  - `Did the Update Work?`
  - `Tell Team: Update Failed`
  - `Email Team — Change Rejected`
  - `Save Change History Record`
  - `Prepare Stakeholder Notification`
  - `Send Alert to Stakeholder`
  - `Email Team — Process Updated`

- **Node Details:**
  - **Did the Manager Approve?**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Conditional Branching)
    - *Configuration Choices:* Evaluates `={{ $json.approved }}`.
    - *Key Expressions/Variables:* `{{ $json.approved }}`
    - *Connections:* Input from approval reader. True branch connects to `Create New Process Version (Keep Old One)`; False branch connects to `Tell Team: Change Was Rejected`.
    - *Potential Failures:* Boolean type resolution mismatch.
  - **Tell Team: Change Was Rejected**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration Choices:* Formats human rejection notification details.
    - *Key Expressions/Variables:* References approval reader data.
    - *Connections:* Input from false branch of manager approval check; output to `Email Team — Change Rejected` (note: labeled as change rejected email).
    - *Potential Failures:* None.
  - **Email Team — Change Rejected**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (Email Integration)
    - *Configuration Choices:* Sends email notification to `info@example.com` informing stakeholders that the proposed change was rejected by management.
    - *Key Expressions/Variables:* Subject: `={{ $json.notification.subject }}`, Message: `={{ $json.notification.body }}`.
    - *Credentials Required:* `gmailOAuth2`
    - *Connections:* Input from `Tell Team: Update Failed` or `Tell Team: Change Was Rejected`. Terminates branch.
    - *Potential Failures:* Authentication errors or invalid recipient configuration.
  - **Create New Process Version (Keep Old One)**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration Choices:* Builds the version record (`newVersion`, `previousVersion`, rules configuration, diagram ASCII generation, and configuration snapshot for rollback).
    - *Key Expressions/Variables:* References approval reader data.
    - *Connections:* Input from true branch of manager approval check; output to `Apply the New Approval Process`.
    - *Potential Failures:* Helper function parsing errors when modifying approval rules.
  - **Apply the New Approval Process**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration Choices:* Simulates in-memory database updates (`workflowConfigurationStore`) with rollback capability and persistence hints.
    - *Key Expressions/Variables:* References version record creation data.
    - *Connections:* Input from version creation node; output to `Did the Update Work?`.
    - *Potential Failures:* Missing version properties or invalid structure.
  - **Did the Update Work?**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Conditional Branching)
    - *Configuration Choices:* Evaluates `={{ $json.updateSuccess }}`.
    - *Key Expressions/Variables:* `{{ $json.updateSuccess }}`
    - *Connections:* Input from application update node. True branch connects to `Save Change History Record`; False branch connects to `Tell Team: Update Failed`.
    - *Potential Failures:* Type mismatch on update success property.
  - **Tell Team: Update Failed**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration Choices:* Formats failure payload for notification alerts.
    - *Key Expressions/Variables:* References application update failure state.
    - *Connections:* Input from false branch of update check; output to `Email Team — Change Rejected` (reuse of email node for failure alerts).
    - *Potential Failures:* None.
  - **Save Change History Record**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration Choices:* Compiles a comprehensive audit log (`auditLog`) including old/new configurations, risk level, AI confidence, and validation status.
    - *Key Expressions/Variables:* References successful update data.
    - *Connections:* Input from true branch of update check; output to `Prepare Stakeholder Notification`.
    - *Potential Failures:* Missing audit metadata.
  - **Prepare Stakeholder Notification**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration Choices:* Formats human-readable Slack and email broadcast messages containing version transition summaries and efficiency gains.
    - *Key Expressions/Variables:* References audit log and configuration store.
    - *Connections:* Input from audit history record; output to `Send Alert to Stakeholder` and `Email Team — Process Updated`.
    - *Potential Failures:* Undefined index lookups.
  - **Send Alert to Stakeholder**
    - *Type and Technical Role:* `n8n-nodes-base.slack` (Messaging Integration)
    - *Configuration Choices:* Posts notification messages to the selected Slack channel (`C09S57E2JQ2`).
    - *Key Expressions/Variables:* Text: `=={{ $json.notification.subject }}\n\n{{ $json.notification.message }}`.
    - *Credentials Required:* `slackApi`
    - *Connections:* Input from stakeholder notification preparation. Terminates branch.
    - *Potential Failures:* Invalid channel ID, token permission scope issues, or rate limiting.
  - **Email Team — Process Updated**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (Email Integration)
    - *Configuration Choices:* Sends broadcast emails to `info@example.com` confirming successful workflow updates.
    - *Key Expressions/Variables:* Subject: `={{ $json.notification.subject }}`, Message: `={{ $json.notification.body }}`.
    - *Credentials Required:* `gmailOAuth2`
    - *Connections:* Input from `Tell Team: Change Was Rejected` or notification preparation. Terminates branch.
    - *Potential Failures:* SMTP authentication errors.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Start Weekly Review | `scheduleTrigger` | Triggers weekly process review | None | Changes Check If Company Policies Changed | Collect Review Inputs |
| Changes Check If Company Policies Changed | `code` | Detects policy version changes | Start Weekly Review | Get Past Purchase Approval History | Collect Review Inputs |
| Get Past Purchase Approval History | `code` | Loads mock procurement transactions | Changes Check If Company Policies Changed | Get Current Approval Steps | Collect Review Inputs |
| Get Current Approval Steps | `code` | Loads sequential approval rules | Get Past Purchase Approval History | Get Company Buying Rules | Collect Review Inputs |
| Get Company Buying Rules | `code` | Loads active compliance policies | Get Current Approval Steps | Clean and Organize the Data | Collect Review Inputs |
| Clean and Organize the Data | `code` | Normalizes transactions and delay metrics | Get Company Buying Rules | Find What Is Slowing Approvals | Analyze Performance and Decide |
| Find What Is Slowing Approvals | `code` | Calculates per-role bottleneck scores | Clean and Organize the Data | Score Overall Approval Performance | Analyze Performance and Decide |
| Score Overall Approval Performance | `code` | Evaluates E2E metrics & optimization need | Find What Is Slowing Approvals | Do We Need to Improve the Process? | Analyze Performance and Decide |
| Do We Need to Improve the Process? | `if` | Conditional branch for optimization | Score Overall Approval Performance | Prepare Information for AI, No Change Needed — Stop Here | Analyze Performance and Decide |
| No Change Needed — Stop Here | `code` | Terminates branch when no change needed | Do We Need to Improve the Process? | None | Analyze Performance and Decide |
| Prepare Information for AI | `code` | Structures context and prompts for AI | Do We Need to Improve the Process? | Use Demo AI or Real AI? | Analyze Performance and Decide |
| Use Demo AI or Real AI? | `if` | Switches between mock and live AI | Prepare Information for AI | Demo AI Suggestion (No Login Needed), Real AI Suggestion (OpenAI) | Get AI Suggestion and Check Rules |
| Demo AI Suggestion (No Login Needed) | `code` | Generates credential-free mock AI JSON | Use Demo AI or Real AI? | Read AI Answer as Structured Data | Get AI Suggestion and Check Rules |
| Real AI Suggestion (OpenAI) | `openAi` | Calls live OpenAI model | Use Demo AI or Real AI? | Read AI Answer as Structured Data | Get AI Suggestion and Check Rules |
| Read AI Answer as Structured Data | `code` | Parses and cleans LLM JSON response | Demo AI Suggestion (No Login Needed), Real AI Suggestion (OpenAI) | Check AI Suggestion Against Company Rules | Get AI Suggestion and Check Rules |
| Check AI Suggestion Against Company Rules | `code` | Validates suggestion against 10 rules | Read AI Answer as Structured Data | Is the AI Suggestion Safe to Use? | Get AI Suggestion and Check Rules |
| Is the AI Suggestion Safe to Use? | `if` | Validates suggestion safety | Check AI Suggestion Against Company Rules | Create Change Request for Approval, Tell Team: AI Suggestion Rejected | Safety Decision and Manager Approval |
| Tell Team: AI Suggestion Rejected | `code` | Formats validation rejection notification | Is the AI Suggestion Safe to Use? | Email Team — AI Suggestion Rejected | Safety Decision and Manager Approval |
| Email Team — AI Suggestion Rejected | `gmail` | Sends email alert for rejected AI suggestion | Tell Team: AI Suggestion Rejected | None | Safety Decision and Manager Approval |
| Create Change Request for Approval | `code` | Creates pending change proposal record | Is the AI Suggestion Safe to Use? | Wait for Manager Approval | Safety Decision and Manager Approval |
| Wait for Manager Approval | `wait` | Pauses workflow for manager webhook approval | Create Change Request for Approval | Read Approve / Reject Answer | Safety Decision and Manager Approval |
| Read Approve / Reject Answer | `code` | Normalizes incoming webhook approval payload | Wait for Manager Approval | Did the Manager Approve? | Safety Decision and Manager Approval |
| Did the Manager Approve? | `if` | Checks manager's approval decision | Read Approve / Reject Answer | Create New Process Version (Keep Old One), Tell Team: Change Was Rejected | Apply Approved Change or Reject It |
| Tell Team: Change Was Rejected | `code` | Formats notification for human rejection | Did the Manager Approve? | Email Team — Process Updated | Apply Approved Change or Reject It |
| Create New Process Version (Keep Old One) | `code` | Generates new version & rollback snapshot | Did the Manager Approve? | Apply the New Approval Process | Apply Approved Change or Reject It |
| Apply the New Approval Process | `code` | Updates in-memory configuration store | Create New Process Version (Keep Old One) | Did the Update Work? | Apply Approved Change or Reject It |
| Did the Update Work? | `if` | Validates configuration update success | Apply the New Approval Process | Save Change History Record, Tell Team: Update Failed | Confirm Update, Save History, and Notify |
| Tell Team: Update Failed | `code` | Formats update failure notification | Did the Update Work? | Email Team — Change Rejected | Confirm Update, Service History, and Notify |
| Save Change History Record | `code` | Saves complete audit trail log | Did the Update Work? | Prepare Stakeholder Notification | Confirm Update, Save History, and Notify |
| Prepare Stakeholder Notification | `code` | Formats Slack and email notifications | Save Change History Record | Send Alert to Stakeholder, Email Team — Process Updated | Confirm Update, Save History, and Notify |
| Send Alert to Stakeholder | `slack` | Sends Slack notification | Prepare Stakeholder Notification | None | Confirm Update, Save History, and Notify |
| Email Team — Process Updated | `gmail` | Sends email for updated process or human rejection | Tell Team: Change Was Rejected, Prepare Stakeholder Notification | None | Confirm Update, Save History, and Notify |
| Email Team — Change Rejected | `gmail` | Sends email for update failure or human rejection | Tell Team: Update Failed, Tell Team: Change Was Rejected | None | Confirm Update, Save History, and Notify |

---

### 4. Reproducing the Workflow from Scratch

Follow these sequential steps to rebuild the workflow entirely within n8n:

1. **Create Schedule Trigger (`Start Weekly Review`)**
   - Add a **Schedule Trigger** node. Set the interval rule to trigger weekly (`weeks`).
2. **Add Policy Version Check (` Changes Check If Company Policies Changed`)**
   - Add a **Code** node connected to the trigger. Add JavaScript to define `lastPolicyVersion = '2.0'`, `currentPolicyVersion = '2.1'`, and evaluate `policyChanged`.
3. **Add Approval History Loader (`Get Past Purchase Approval History`)**
   - Add a **Code** node. Populate it with the mock historical array of 25 transactions with role durations and statuses.
4. **Add Current Rules Loader (`Get Current Approval Steps`)**
   - Add a **Code** node. Define sequential approval rules (`RULE-001` to `RULE-003`) by amount bands and allowed operational roles.
5. **Add Buying Rules Loader (`Get Company Buying Rules`)**
   - Add a **Code** node to define active policies (high-value controls, manager accountability, compliance integrity).
6. **Add Data Normalizer (`Clean and Organize the Data`)**
   - Add a **Code** node. Normalize transactions and approval events against a 480-minute delay threshold.
7. **Add Bottleneck Analyzer (`Find What Is Slowing Approvals`)**
   - Add a **Code** node. Compute per-role metrics (average duration, delay rate, rejection rate, volume) and calculate normalized `bottleneckScore`.
8. **Add Performance Scorer (`Score Overall Approval Performance`)**
   - Add a **Code** node. Aggregate E2E duration, rejection/rework rates, and evaluate the `optimizationNeeded` boolean flag.
9. **Add Optimization Branch (`Do We Need to Improve the Process?`)**
   - Add an **If** node. Condition: `={{ $json.optimizationNeeded }}` equals `true`.
   - Connect the False branch to a **Code** node (`No Change Needed — Stop Here`).
10. **Add AI Context Preparation (`Prepare Information for AI`)**
    - Add a **Code** node to compile transaction patterns, mandatory roles, system prompts, and user prompts.
11. **Add AI Router (`Use Demo AI or Real AI?`)**
    - Add an **If** node. Condition: `={{ $json.useMockAi }}` equals `false`.
    - Connect True branch to a **Code** node (`Demo AI Suggestion (No Login Needed)`).
    - Connect False branch to an **OpenAI** node (`Real AI Suggestion (OpenAI)` using model `gpt-4.1` with `openAiApi` credentials).
12. **Add JSON Parser (`Read AI Answer as Structured Data`)**
    - Add a **Code** node to extract, strip markdown fences, and parse LLM output into structured JSON.
13. **Add Rule Validator (`Check AI Suggestion Against Company Rules`)**
    - Add a **Code** node to validate the recommendation against 10 deterministic compliance and safety rules.
14. **Add Safety Gate (`Is the AI Suggestion Safe to Use?`)**
    - Add an **If** node. Condition: `={{ $json.valid }}` equals `true`.
    - Connect False branch to a **Code** node (`Tell Team: AI Suggestion Rejected`) followed by a **Gmail** node (`Email Team — AI Suggestion Rejected` using `gmailOAuth2` credentials).
15. **Add Change Request & Wait Node (`Create Change Request for Approval` & `Wait for Manager Approval`)**
    - Add a **Code** node (`Create Change Request for Approval`) to generate a unique change ID and proposed version (`v4`).
    - Add a **Wait** node (`Wait for Manager Approval`). Set resume method to `Webhook`, HTTP method to `POST`, and note the generated webhook ID.
16. **Add Approval Parser (`Read Approve / Reject Answer`)**
    - Add a **Code** node to normalize webhook payloads and check for `additionalApproval` requirements on high-risk changes.
17. **Add Manager Approval Branch (`Did the Manager Approve?`)**
    - Add an **If** node. Condition: `={{ $json.approved }}` equals `true`.
    - Connect False branch to a **Code** node (`Tell Team: Change Was Rejected`) linked to a **Gmail** node (`Email Team — Change Rejected`).
18. **Add Versioning & Application Logic (`Create New Process Version (Keep Old One)` & `Apply the New Approval Process`)**
    - Add a **Code** node (`Create New Process Version (Keep Old One)`) to generate version records and rollback snapshots.
    - Add a **Code** node (`Apply the New Approval Process`) to update the in-memory configuration store.
19. **Add Update Validation & Audit (`Did the Update Work?`, `Tell Team: Update Failed`, & `Save Change History Record`)**
    - Add an **If** node (`Did the Update Work?`) checking `={{ $json.updateSuccess }}`.
    - Connect False branch to a **Code** node (`Tell Team: Update Failed`) linked to the rejection email node.
    - Connect True branch to a **Code** node (`Save Change History Record`) to generate the deployment audit log.
20. **Add Broadcast & Notifications (`Prepare Stakeholder Notification`, `Send Alert to Stakeholder`, & `Email Team — Process Updated`)**
    - Add a **Code** node (`Prepare Stakeholder Notification`) to format stakeholder messages.
    - Add a **Slack** node (`Send Alert to Stakeholder` using `slackApi` credentials) configured with target channel ID `C09S57E2JQ2`.
    - Add a **Gmail** node (`Email Team — Process Updated` using `gmailOAuth2` credentials) sending confirmation emails to `info@example.com`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Adaptive Procurement Workflow Designer (Procurement Industry) | Complete automated weekly audit workflow reviewing purchase approvals, detecting delays (typically Finance), and generating compliant AI optimizations. |
| Setup and Testing Instructions | 1. Import workflow. 2. Keep Demo AI enabled for testing or configure OpenAI credentials. 3. Connect Slack and Gmail credentials. 4. Execute test run. 5. Send approval payload via POST to the wait node resume webhook URL. 6. Confirm version creation and stakeholder alerts. |
Triage purchase requisitions with Airtable, Groq, Google Sheets, and Slack

https://n8nworkflows.xyz/workflows/triage-purchase-requisitions-with-airtable--groq--google-sheets--and-slack-20534


# Triage purchase requisitions with Airtable, Groq, Google Sheets, and Slack

### 1. Workflow Overview

The **Urgent Requisition Auto-Triage Workflow** serves as an automated procurement triage engine. Its primary purpose is to process incoming purchase requisitions from Airtable, evaluate business urgency through a combination of LLM-based intent analysis and strict keyword scanning, cross-reference department-specific rules and budget thresholds, and route requests to the correct operational tier (VP approval, auto fast-track, or the standard queue). Finally, it logs an immutable audit trail to Google Sheets, updates the source record status in Airtable, and dispatches team notifications via Slack.

The workflow is structured into the following logical blocks:
- **1.1 Input Reception & Formatting:** Captures newly submitted purchase requisitions and prepares the raw request text for AI evaluation.
- **1.2 AI Processing & Urgency Evaluation:** Sends the formatted description to a Groq-hosted Large Language Model (LLM) to assess business impact and output a structured JSON urgency rating.
- **1.3 Priority Scoring Engine:** Scans request text for critical operational keywords, fetches departmental routing parameters from Airtable, and calculates a final mathematical priority score and routing path.
- **1.4 Pathway Routing:** Evaluates the computed routing decision and directs the unified execution payload into the appropriate processing tier.
- **1.5 Execution & Audit Logging:** Updates the source record in Airtable, records a timestamped audit entry in Google Sheets, and sends targeted Slack alerts for escalations and fast-tracks.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Reception & Formatting
- **Overview:** This block triggers automatically upon the creation of a new purchase requisition record in Airtable and structures the description field into an isolated variable string for downstream processing.
- **Nodes Involved:** 
  - `Trigger: New PR`
  - `Format AI Prompt`
- **Node Details:**
  - **`Trigger: New PR`**
    - *Type & Technical Role:* `n8n-nodes-base.airtableTrigger` (Polling Trigger)
    - *Configuration Choices:* Configured to poll the target Airtable base (`appiNvQQaSswkQ13m`) and table (`tblfuDLABXuBEZiK7`) every minute, triggering when the `submitted_at` date/time field is populated.
    - *Key Expressions/Variables:* None (entry point).
    - *Input/Output:* Inputs: None (External Webhook/Polling). Outputs: Connects to `Format AI Prompt`.
    - *Version-specific requirements:* Uses Airtable API token authentication.
    - *Edge Cases/Failure Types:* Polling delays or API authentication revocation.
  - **`Format AI Prompt`**
    - *Type & Technical Role:* `n8n-nodes-base.set` (Data Transformation)
    - *Configuration Choices:* Assigns the incoming Airtable description field to a clean variable named `prompt_text`.
    - *Key expressions:* `={{ $json.fields.description }}`
    - *Input/Output:* Input: `Trigger: New PR`. Output: Connects to `LLM: Evaluate Urgency`.

#### Block 1.2: AI Processing & Urgency Evaluation
- **Overview:** Leverages a Groq-hosted LLM combined with a strict output schema parser to analyze the request text, determine business impact, and return a standardized integer score from 1 to 5 with supporting reasoning.
- **Nodes Involved:**
  - `LLM: Evaluate Urgency`
  - `Model: Groq (gpt-oss-20b)`
  - `Parse: JSON Output`
  - `Extract AI Score`
- **Node Details:**
  - **`LLM: Evaluate Urgency`**
    - *Type & Technical Role:* `@n8n/n8n-nodes-langchain.chainLlm` (AI Chain)
    - *Configuration Choices:* Uses system prompt instructions to act as a strict procurement triage assistant, scoring descriptions from 1 (routine) to 5 (emergency) while explicitly ignoring user-injected override demands.
    - *Key expressions:* `={{ $json.prompt_text }}`
    - *Input/Output:* Inputs: `Format AI Prompt` (main), `Model: Groq` (AI language model), `Parse: JSON Output` (AI output parser). Output: Connects to `Extract AI Score`.
  - **`Model: Groq (gpt-oss-20b)`**
    - *Type & Technical Role:* `@n8n/n8n-nodes-langchain.lmChatGroq` (Chat Model Provider)
    - *Configuration Choices:* Configured to use the `openai/gpt-oss-20b` model.
    - *Credentials:* Groq API credential (`Groq (Mihir-Gmail)`).
    - *Input/Output:* Output: Connects to `LLM: Evaluate Urgency`.
  - **`Parse: JSON Output`**
    - *Type & Technical Role:* `@n8n/n8n-nodes-langchain.outputParserStructured` (Structured Output Parser)
    - *Configuration Choices:* Enforces a strict JSON schema containing required integer property `ai_score` and string property `reasoning`, with `additionalProperties: false`.
    - *Input/Output:* Output: Connects to `LLM: Evaluate Urgency`.
  - **`Extract AI Score`**
    - *Type & Technical Role:* `n8n-nodes-base.set` (Data Transformation)
    - *Configuration Choices:* Extracts parsed JSON attributes into top-level node properties for subsequent JavaScript calculations.
    - *Key expressions:* `ai_score`: `={{ $json.output.ai_score }}`, `reasoning`: `={{ $json.output.reasoning }}`
    - *Input/Output:* Input: `LLM: Evaluate Urgency`. Output: Connects to `Scan: Critical Keywords`.

#### Block 1.3: Priority Scoring Engine
- **Overview:** Scans description and justification text for high-priority operational keywords, fetches departmental governance rules from a reference Airtable base, and computes a final blended priority score and routing path.
- **Nodes Involved:**
  - `Scan: Critical Keywords`
  - `Fetch: Dept Rules`
  - `Calculate Final Route`
- **Node Details:**
  - **`Scan: Critical Keywords`**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Custom JavaScript Execution)
    - *Configuration Choices:* Searches combined lowercase description and business justification text for arrays of critical keywords (e.g., "outage", "breach") and urgent keywords (e.g., "degraded", "blocker"). Assigns `keyword_urgency_score` as 5 (critical), 3.5 (urgent), or 1 (default).
    - *Input/Output:* Input: `Extract AI Score`. Output: Connects to `Fetch: Dept Rules`.
  - **`Fetch: Dept Rules`**
    - *Type & Technical Role:* `n8n-nodes-base.airtable` (Database Read / Search)
    - *Configuration Choices:* Queries the department rules table (`tbljY94pR46RDSH6W`) in base `appSauheyM8z11SEk`, filtering records where `{department_name}` matches the requester department from the trigger node.
    - *Key expressions:* `={department_name} = '{{ $('Trigger: New PR').item.json.fields.requester_department }}'`
    - *Input/Output:* Input: `Scan: Critical Keywords`. Output: Connects to `Calculate Final Route`.
  - **`Calculate Final Route`**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Custom JavaScript Execution)
    - *Configuration Choices:* Combines LLM intent score and keyword score (`Math.max`), multiplies by department baseline criticality, evaluates estimated cost against department VP thresholds, and establishes final routing decisions (`vp_approval`, `fast_track`, or `fallback`).
    - *Input/Output:* Input: `Fetch: Dept Rules`. Output: Connects to `Router: PR Tiers`.

#### Block 1.4: Pathway Routing
- **Overview:** Evaluates the computed routing decision property and splits workflow execution into dedicated processing branches.
- **Nodes Involved:**
  - `Router: PR Tiers`
- **Node Details:**
  - **`Router: PR Tiers`**
    - *Type & Technical Role:* `n8n-nodes-base.switch` (Conditional Router)
    - *Configuration Choices:* Routes items based on string equality of `{{ $json.routing_decision }}`:
      - Branch 0: Matches `vp_approval`
      - Branch 1: Matches `fast_track`
      - Fallback Output: Standard queue routing.
    - *Input/Output:* Input: `Calculate Final Route`. Outputs: Connects execution paths to logging and status update nodes.

#### Block 1.5: Execution & Audit Logging
- **Overview:** Persists status changes to the source Airtable record, logs execution metrics to Google Sheets, and sends notifications to configured Slack channels.
- **Nodes Involved:**
  - `Log: Google Sheets Audit`
  - `Update: VP Pending`
  - `Update: Fast-Tracked`
  - `Update: Standard Queue`
  - `Send a message` (VP Escalation)
  - `Send a message1` (Fast-Track Alert)
- **Node Details:**
  - **`Log: Google Sheets Audit`**
    - *Type & Technical Role:* `n8n-nodes-base.googleSheets` (Spreadsheet Append/Update)
    - *Configuration Choices:* Appends or updates rows in Google Sheet `1Y66owon9R95quVsKYuQz_sfqi_nyw7MAgTiT67cPucY` using `PR_ID` as the matching column.
    - *Key expressions:* Maps cost, PR ID, timestamp (`{{$now}}`), department, routing reasons, and scores.
    - *Input/Output:* Input: Connected to all output branches of `Router: PR Tiers`.
  - **`Update: VP Pending`**
    - *Type & Technical Role:* `n8n-nodes-base.airtable` (Database Update)
    - *Configuration Choices:* Updates the matching Airtable record status to `Pending` and writes routing notes and final scores.
    - *Input/Output:* Input: `Router: PR Tiers` (Branch 0). Output: Connects to `Send a message`.
  - **`Update: Fast-Tracked`**
    - *Type & Technical Role:* `n8n-nodes-base.airtable` (Database Update)
    - *Configuration Choices:* Updates the matching Airtable record status to `Auto Fast-Tracked.` and writes routing notes and final scores.
    - *Input/Output:* Input: `Router: PR Tiers` (Branch 1). Output: Connects to `Send a message1`.
  - **`Update: Standard Queue`**
    - *Type & Technical Role:* `n8n-nodes-base.airtable` (Database Update)
    - *Configuration Choices:* Updates the matching Airtable record status to `Pending` for standard queue entries.
    - *Input/Output:* Input: `Router: PR Tiers` (Fallback Branch). Output: None (terminal branch).
  - **`Send a message`** (VP Escalation)
    - *Type & Technical Role:* `n8n-nodes-base.slack` (Messaging Notification)
    - *Configuration Choices:* Dispatches Slack notifications for VP approval requirements.
    - *Input/Output:* Input: `Update: VP Pending`. Output: None (terminal branch).
  - **`Send a message1`** (Fast-Track Alert)
    - *Type & Technical Role:* `n8n-nodes-base.slack` (Messaging Notification)
    - *Configuration Choices:* Dispatches Slack notifications for auto fast-tracked items.
    - *Input/Output:* Input: `Update: Fast-Tracked`. Output: None (terminal branch).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | `n8n-nodes-base.stickyNote` | Documentation note | None | None | ## Data Ingestion<br>Captures the initial purchase requisition from Airtable upon submission and formats the raw description string for the AI to analyze. |
| Trigger: New PR | `n8n-nodes-base.airtableTrigger` | Polling Trigger | None | Format AI Prompt | ## Data Ingestion<br>Captures the initial purchase requisition from Airtable upon submission and formats the raw description string for the AI to analyze. |
| Format AI Prompt | `n8n-nodes-base.set` | Data Transformation | Trigger: New PR | LLM: Evaluate Urgency | ## Data Ingestion<br>Captures the initial purchase requisition from Airtable upon submission and formats the raw description string for the AI to analyze. |
| Sticky Note1 | `n8n-nodes-base.stickyNote` | Documentation note | None | None | ## AI Urgency Evaluation<br>Processes the description through an LLM to evaluate the business impact, generating a structured JSON urgency score and reasoning context. |
| LLM: Evaluate Urgency | `@n8n/n8n-nodes-langchain.chainLlm` | AI Chain | Format AI Prompt, Model: Groq (gpt-oss-20b), Parse: JSON Output | Extract AI Score | ## AI Urgency Evaluation<br>Processes the description through an LLM to evaluate the business impact, generating a structured JSON urgency score and reasoning context. |
| Model: Groq (gpt-oss-20b) | `@n8n/n8n-nodes-langchain.lmChatGroq` | Chat Model Provider | None | LLM: Evaluate Urgency | ## AI Urgency Evaluation<br>Processes the description through an LLM to evaluate the business impact, generating a structured JSON urgency score and reasoning context. |
| Parse: JSON Output | `@n8n/n8n-nodes-langchain.outputParserStructured` | Output Parser | None | LLM: Evaluate Urgency | ## AI Urgency Evaluation<br>Processes the description through an LLM to evaluate the business impact, generating a structured JSON urgency score and reasoning context. |
| Extract AI Score | `n8n-nodes-base.set` | Data Transformation | LLM: Evaluate Urgency | Scan: Critical Keywords | ## AI Urgency Evaluation<br>Processes the description through an LLM to evaluate the business impact, generating a structured JSON urgency score and reasoning context. |
| Sticky Note2 | `n8n-nodes-base.stickyNote` | Documentation note | None | None | ## Priority Scoring Engine<br>Scans for critical keywords, retrieves department-specific rules, blends scores, and runs the final math to determine the exact routing path. |
| Scan: Critical Keywords | `n8n-nodes-base.code` | Custom JavaScript | Extract AI Score | Fetch: Dept Rules | ## Priority Scoring Engine<br>Scans for critical keywords, retrieves department-specific rules, blends scores, and runs the final math to determine the exact routing path. |
| Fetch: Dept Rules | `n8n-nodes-base.airtable` | Database Read / Search | Scan: Critical Keywords | Calculate Final Route | ## Priority Scoring Engine<br>Scans for critical keywords, retrieves department-specific rules, blends scores, and runs the final math to determine the exact routing path. |
| Calculate Final Route | `n8n-nodes-base.code` | Custom JavaScript | Fetch: Dept Rules | Router: PR Tiers | ## Priority Scoring Engine<br>Scans for critical keywords, retrieves department-specific rules, blends scores, and runs the final math to determine the exact routing path. |
| Sticky Note3 | `n8n-nodes-base.stickyNote` | Documentation note | None | None | ## Pathway Routing<br>Evaluates the final computed routing decision and directs the unified payload into the correct operational tier for execution or escalation. |
| Router: PR Tiers | `n8n-nodes-base.switch` | Conditional Router | Calculate Final Route | Log: Google Sheets Audit, Update: VP Pending, Update: Fast-Tracked, Update: Standard Queue | ## Pathway Routing<br>Evaluates the final computed routing decision and directs the unified payload into the correct operational tier for execution or escalation. |
| Sticky Note4 | `n8n-nodes-base.stickyNote` | Documentation note | None | None | ## Execution & Audit Logging<br>Updates the original database record with the new status, logs the full audit trail in Sheets, and dispatches Slack notifications. |
| Log: Google Sheets Audit | `n8n-nodes-base.googleSheets` | Spreadsheet Append/Update | Router: PR Tiers | None | ## Execution & Audit Logging<br>Updates the original database record with the new status, logs the full audit trail in Sheets, and dispatches Slack notifications. |
| Update: VP Pending | `n8n-nodes-base.airtable` | Database Update | Router: PR Tiers | Send a message | ## Execution & Audit Logging<br>Updates the original database record with the new status, logs the full audit trail in Sheets, and dispatches Slack notifications. |
| Update: Fast-Tracked | `n8n-nodes-base.airtable` | Database Update | Router: PR Tiers | Send a message1 | ## Execution & Audit Logging<br>Updates the original database record with the new status, logs the full audit trail in Sheets, and dispatches Slack notifications. |
| Update: Standard Queue | `n8n-nodes-base.airtable` | Database Update | Router: PR Tiers | None | ## Execution & Audit Logging<br>Updates the original database record with the new status, logs the full audit trail in Sheets, and dispatches Slack notifications. |
| Sticky Note5 | `n8n-nodes-base.stickyNote` | Documentation note | None | None | # Workflow Overview: Urgent Requisition Auto-Triage<br><br>This workflow acts as an automated **Procurement Triage Engine** for purchase requisitions... |
| Send a message | `n8n-nodes-base.slack` | Messaging Notification | Update: VP Pending | None | ## Execution & Audit Logging<br>Updates the original database record with the new status, logs the full audit trail in Sheets, and dispatches Slack notifications. |
| Send a message1 | `n8n-nodes-base.slack` | Messaging Notification | Update: Fast-Tracked | None | ## Execution & Audit Logging<br>Updates the original database record with the new status, logs the full audit trail in Sheets, and dispatches Slack notifications. |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow manually in n8n:

1. **Create the Airtable Trigger (`Trigger: New PR`):**
   - Add an **Airtable Trigger** node.
   - Configure authentication using an Airtable Personal Access Token credential.
   - Set Base ID to `appiNvQQaSswkQ13m`, Table ID to `tblfuDLABXuBEZiK7`, and Poll Times to run every minute.
   - Set `submitted_at` as the Trigger Field.

2. **Format the Prompt (`Format AI Prompt`):**
   - Add a **Set** node connected to the trigger output.
   - Create a string assignment named `prompt_text` with value `={{ $json.fields.description }}`.

3. **Configure the AI Chain and Model:**
   - Add an **Advanced AI Chain (`LLM: Evaluate Urgency`)** node connected to `Format AI Prompt`.
   - Set the prompt text input to `={{ $json.prompt_text }}`.
   - Configure the system message in the node parameters to act as a strict procurement triage assistant with a 1–5 scoring guide and JSON output rule.
   - Connect a **Groq Chat Model (`Model: Groq (gpt-oss-20b)`)** node to the AI language model input port. Set model to `openai/gpt-oss-20b` and attach a valid Groq API credential.
   - Connect a **Structured Output Parser (`Parse: JSON Output`)** node to the AI output parser port. Define a manual JSON schema requiring integer `ai_score` (1–5) and string `reasoning`.

4. **Extract AI Results (`Extract AI Score`):**
   - Add a **Set** node connected to the output of `LLM: Evaluate Urgency`.
   - Create number assignment `ai_score` = `={{ $json.output.ai_score }}` and string assignment `reasoning` = `={{ $json.output.reasoning }}`.

5. **Scan Keywords (`Scan: Critical Keywords`):**
   - Add a Code node using JavaScript mode.
   - Insert logic to scan trigger description and business justification fields for critical keywords (`outage`, `downtime`, `production down`, `sev-1`, `sev 1`, `breach`, `data loss`) and urgent keywords (`degraded`, `capacity limit`, `security risk`, `blocker`, `compliance deadline`, `imminent failure`), assigning `keyword_urgency_score` (5, 3.5, or 1).

6. **Fetch Department Rules (`Fetch: Dept Rules`):**
   - Add an Airtable node set to **Search** operation.
   - Set Base to `appSauheyM8z11SEk` and Table to `tbljY94pR46RDSH6W`.
   - Set Filter by Formula to `={department_name} = '{{ $('Trigger: New PR').item.json.fields.requester_department }}'`.

7. **Calculate Final Scores and Routes (`Calculate Final Route`):**
   - Add a Code node using JavaScript mode.
   - Implement logic to calculate `blended_intent_score = Math.max(llmScore, keywordScore)`, compute `final_score = blended_intent_score * deptCriticality`, and establish routing decisions based on cost thresholds and fast-track eligibility. Output unified JSON properties.

8. **Route Tiers (`Router: PR Tiers`):**
   - Add a **Switch** node.
   - Define rule 0: String equals `vp_approval`.
   - Define rule 1: String equals `fast_track`.
   - Enable fallback output for the standard queue.

9. **Audit Logging & Airtable Updates:**
   - Add a **Google Sheets** node (`Log: Google Sheets Audit`) set to `appendOrUpdate` using `PR_ID` as the matching column. Connect all three outputs of `Router: PR Tiers` to this node.
   - Add three **Airtable** nodes (`Update: VP Pending`, `Update: Fast-Tracked`, `Update: Standard Queue`) set to **Update** operation using `pr_id` as the matching column, mapping status, routing notes, and final priority scores respectively to each branch.

10. **Dispatch Slack Notifications:**
    - Connect `Update: VP Pending` to a **Slack** node configured to send a message to the executive escalation channel.
    - Connect `Update: Fast-Tracked` to a **Slack** node configured to send a message to the fast-track channel.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Professional n8n development & custom enterprise automation support | [WeblineIndia](https://www.weblineindia.com/) |
| Workflow source industry context | Procurement Triage & ERP Readiness |
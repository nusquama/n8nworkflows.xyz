Generate daily multi-cloud FinOps cost and carbon reports with OpenAI GPT-4o

https://n8nworkflows.xyz/workflows/generate-daily-multi-cloud-finops-cost-and-carbon-reports-with-openai-gpt-4o-14003


# Generate daily multi-cloud FinOps cost and carbon reports with OpenAI GPT-4o

# 1. Workflow Overview

This workflow generates a daily multi-cloud FinOps analysis report by collecting billing export data, parsing it, passing it through a supervised multi-agent AI architecture, and formatting the final result into a structured report payload.

Its main purpose is to help FinOps, cloud operations, DevOps, and executive stakeholders monitor:

- cloud cost trends across Azure, AWS, and GCP
- resource underutilization and waste
- optimization opportunities
- estimated carbon footprint and sustainability improvements
- executive-ready financial recommendations

The workflow is organized into the following logical blocks.

## 1.1 Scheduled Input Reception and Billing Retrieval

A daily schedule trigger starts the workflow at 02:00. It then performs an HTTP request to retrieve billing export data from an internal or external billing endpoint and parses that file into structured data.

## 1.2 Central AI Orchestration

The parsed billing data is sent to a central LangChain-based AI agent acting as the multi-cloud optimization supervisor. This orchestrator uses GPT-4o as its primary reasoning model and coordinates specialist sub-agents and tools.

## 1.3 Specialized Analysis Agents

Four specialist AI tool-agents perform domain-specific analysis:

- Resource Utilization Analyzer
- Cost Optimization Agent
- Carbon Footprint Analysis Agent
- FinOps Narrative Generator

These are attached as callable tools for the orchestrator rather than running independently in the main execution chain.

## 1.4 Shared Computational and Output Control Layer

The orchestrator also has access to:

- a calculator tool for numerical computations
- a Python code tool for advanced analytics and modeling
- a structured output parser enforcing a defined JSON schema

This ensures the generated result is machine-readable and consistent.

## 1.5 Final Report Formatting

The orchestrator’s structured output is wrapped into a final report object containing metadata such as generation timestamp, report type, analysis period, and completion status.

---

# 2. Block-by-Block Analysis

## 2.1 Scheduled Input Reception and Billing Retrieval

### Overview

This block starts the workflow automatically every day and retrieves the raw billing export file. The file is then parsed so it can be consumed by the AI orchestration layer.

### Nodes Involved

- Daily Cost Analysis Trigger
- Fetch Billing Exports
- Parse Billing Data

### Node Details

#### Daily Cost Analysis Trigger

- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`  
  Starts the workflow automatically on a recurring schedule.
- **Configuration choices:**  
  Configured with an interval rule that triggers at hour `2`, meaning a daily run at approximately 02:00 server/workflow timezone.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  No input node. Outputs to **Fetch Billing Exports**.
- **Version-specific requirements:**  
  Uses `typeVersion 1.3`. Standard n8n schedule trigger behavior applies.
- **Edge cases or potential failure types:**  
  - Timezone misunderstandings if server timezone differs from business timezone
  - Workflow not running if inactive
  - Missed triggers if the n8n instance is unavailable at schedule time
- **Sub-workflow reference:**  
  None.

#### Fetch Billing Exports

- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Downloads the billing export file from a configured endpoint.
- **Configuration choices:**  
  - URL is a placeholder: `<__PLACEHOLDER_VALUE__internal_billing_export_endpoint__>`
  - Response format is configured as **file**, so the node stores the fetched content as binary data rather than plain JSON/text
- **Key expressions or variables used:**  
  None in the current configuration.
- **Input and output connections:**  
  Input from **Daily Cost Analysis Trigger**. Output to **Parse Billing Data**.
- **Version-specific requirements:**  
  Uses `typeVersion 4.4`. File-response handling should be supported in modern n8n versions.
- **Edge cases or potential failure types:**  
  - Invalid or unreachable billing endpoint
  - Authentication or authorization failure if the endpoint requires credentials but none are configured
  - Non-file response returned by the endpoint
  - Large file download causing timeout or memory pressure
  - CSV export not matching expected parser format
- **Sub-workflow reference:**  
  None.

#### Parse Billing Data

- **Type and technical role:** `n8n-nodes-base.extractFromFile`  
  Extracts structured data from the downloaded file, likely intended for CSV or similar billing export content.
- **Configuration choices:**  
  No special options are defined, so default extraction behavior is used.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  Input from **Fetch Billing Exports**. Output to **Multi-Cloud Optimization Orchestrator**.
- **Version-specific requirements:**  
  Uses `typeVersion 1.1`.
- **Edge cases or potential failure types:**  
  - Unsupported file type
  - Corrupt or malformed CSV/file content
  - Header mismatch or unexpected delimiter
  - Binary property naming issues if HTTP response binary metadata differs from parser expectations
- **Sub-workflow reference:**  
  None.

---

## 2.2 Central AI Orchestration

### Overview

This block is the workflow’s core reasoning engine. It receives the parsed billing dataset and uses a GPT-4o-powered orchestrator agent to coordinate specialized analysis tools and produce a structured final optimization report.

### Nodes Involved

- Multi-Cloud Optimization Orchestrator
- Orchestrator Model
- Structured Output Parser

### Node Details

#### Multi-Cloud Optimization Orchestrator

- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  Main AI supervisor agent that processes the billing data and calls specialized tool-agents and utility tools.
- **Configuration choices:**  
  - Input text is set to `={{ $json }}`, meaning the entire parsed billing record is passed into the agent as the prompt input
  - Custom system message defines the role as a multi-cloud cost optimization orchestrator coordinating four specialist agents
  - `hasOutputParser` is enabled, so output is expected to be validated/formatted via the attached structured parser
- **Key expressions or variables used:**  
  - `={{ $json }}`
- **Input and output connections:**  
  Main input from **Parse Billing Data**  
  AI language model input from **Orchestrator Model**  
  AI tool inputs from:
  - **Resource Utilization Analyzer Agent**
  - **Cost Optimization Agent**
  - **Carbon Footprint Analysis Agent**
  - **FinOps Narrative Generator Agent**
  - **Financial Calculator**
  - **Advanced Analytics Code Tool**  
  AI output parser input from **Structured Output Parser**  
  Main output to **Format Final Report**
- **Version-specific requirements:**  
  Uses `typeVersion 3.1`. Requires n8n setup with LangChain AI nodes available.
- **Edge cases or potential failure types:**  
  - Model/tool invocation loops or weak tool selection
  - Very large billing payload causing token limit issues
  - Parsed data shape too complex for direct prompt injection
  - Structured output schema mismatch causing parser failure
  - Missing OpenAI credentials on linked model node
  - Incomplete downstream result if the agent does not populate required schema fields
- **Sub-workflow reference:**  
  None. This is not a sub-workflow node; it is the parent orchestrator.

#### Orchestrator Model

- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Supplies the language model used by the orchestrator agent.
- **Configuration choices:**  
  - Model: `gpt-4o`
  - Temperature: `0.2` for relatively controlled but still flexible orchestration
  - OpenAI credentials attached
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  Connected to **Multi-Cloud Optimization Orchestrator** as `ai_languageModel`.
- **Version-specific requirements:**  
  Uses `typeVersion 1.3`.
- **Edge cases or potential failure types:**  
  - Invalid OpenAI credentials
  - Model availability or quota issues
  - Rate limits
  - API errors or timeout
- **Sub-workflow reference:**  
  None.

#### Structured Output Parser

- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`  
  Enforces a manual JSON schema on the orchestrator’s final response.
- **Configuration choices:**  
  - Schema type: manual
  - Requires a top-level object containing fields such as:
    - `executive_summary`
    - `current_state`
    - `optimization_opportunities`
    - `six_month_projection`
    - optional/extended sections including reserved strategy, carbon footprint, and action items
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  Connected to **Multi-Cloud Optimization Orchestrator** as `ai_outputParser`.
- **Version-specific requirements:**  
  Uses `typeVersion 1.3`.
- **Edge cases or potential failure types:**  
  - Missing required fields
  - Wrong data types, especially numeric fields returned as strings
  - Nested object mismatch
  - Hallucinated structure not conforming to schema
- **Sub-workflow reference:**  
  None.

---

## 2.3 Resource Utilization Analysis Block

### Overview

This block provides the orchestrator with a specialist tool-agent focused on infrastructure usage efficiency. It identifies over-provisioned, idle, or underutilized resources across cloud providers.

### Nodes Involved

- Resource Utilization Analyzer Agent
- Utilization Analyzer Model

### Node Details

#### Resource Utilization Analyzer Agent

- **Type and technical role:** `@n8n/n8n-nodes-langchain.agentTool`  
  A callable AI tool-agent invoked by the orchestrator when utilization analysis is needed.
- **Configuration choices:**  
  - Tool input text: `={{ $fromAI('billing_data', 'Billing exports and utilization logs to analyze') }}`
  - System message defines detailed utilization analysis tasks across Azure, AWS, and GCP
  - Tool description explains the scope for the supervisor agent
- **Key expressions or variables used:**  
  - `$fromAI('billing_data', 'Billing exports and utilization logs to analyze')`
- **Input and output connections:**  
  AI language model input from **Utilization Analyzer Model**  
  Exposed as an `ai_tool` to **Multi-Cloud Optimization Orchestrator**
- **Version-specific requirements:**  
  Uses `typeVersion 3`.
- **Edge cases or potential failure types:**  
  - The orchestrator may not supply a meaningful `billing_data` payload
  - Billing exports may lack utilization metrics, causing speculative analysis
  - Token overload if raw billing data is too large
  - Inconsistent provider fields preventing provider-level comparison
- **Sub-workflow reference:**  
  None.

#### Utilization Analyzer Model

- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  GPT-4o model for the utilization analysis tool-agent.
- **Configuration choices:**  
  - Model: `gpt-4o`
  - Temperature: `0.1` for more deterministic analytical output
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  Connected to **Resource Utilization Analyzer Agent** as `ai_languageModel`.
- **Version-specific requirements:**  
  Uses `typeVersion 1.3`.
- **Edge cases or potential failure types:**  
  Same as other OpenAI model nodes: auth, quota, rate limit, timeout.
- **Sub-workflow reference:**  
  None.

---

## 2.4 Cost Optimization and Financial Strategy Block

### Overview

This block helps the orchestrator convert usage findings into financial recommendations. It focuses on savings modeling, reserved-vs-on-demand strategies, commitment planning, and cost-benefit analysis.

### Nodes Involved

- Cost Optimization Agent
- Cost Optimization Model
- Financial Calculator
- Advanced Analytics Code Tool

### Node Details

#### Cost Optimization Agent

- **Type and technical role:** `@n8n/n8n-nodes-langchain.agentTool`  
  Callable AI tool-agent for strategic cost analysis.
- **Configuration choices:**  
  - Tool input text: `={{ $fromAI('utilization_findings', 'Resource utilization analysis results to optimize') }}`
  - System message instructs the tool to produce cost modeling, RI/Savings Plan recommendations, ROI estimates, and carbon-aware financial analysis
- **Key expressions or variables used:**  
  - `$fromAI('utilization_findings', 'Resource utilization analysis results to optimize')`
- **Input and output connections:**  
  AI language model input from **Cost Optimization Model**  
  Exposed as an `ai_tool` to **Multi-Cloud Optimization Orchestrator**
- **Version-specific requirements:**  
  Uses `typeVersion 3`.
- **Edge cases or potential failure types:**  
  - If upstream utilization findings are weak, financial outputs may be generic
  - ROI calculations may be approximate unless the orchestrator explicitly uses calculator/code tools
  - Cost history may be insufficient for six-month trajectory accuracy
- **Sub-workflow reference:**  
  None.

#### Cost Optimization Model

- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`
- **Configuration choices:**  
  - Model: `gpt-4o`
  - Temperature: `0.2`
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  Connected to **Cost Optimization Agent** as `ai_languageModel`.
- **Version-specific requirements:**  
  Uses `typeVersion 1.3`.
- **Edge cases or potential failure types:**  
  OpenAI credential, quota, timeout, or rate-limit issues.
- **Sub-workflow reference:**  
  None.

#### Financial Calculator

- **Type and technical role:** `@n8n/n8n-nodes-langchain.toolCalculator`  
  Numeric utility tool available to the orchestrator for precise arithmetic.
- **Configuration choices:**  
  No parameters configured; default calculator behavior.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  Exposed as `ai_tool` to **Multi-Cloud Optimization Orchestrator**.
- **Version-specific requirements:**  
  Uses `typeVersion 1`.
- **Edge cases or potential failure types:**  
  - The orchestrator may underuse or misuse the calculator
  - Inputs must be properly formed expressions/numbers
- **Sub-workflow reference:**  
  None.

#### Advanced Analytics Code Tool

- **Type and technical role:** `@n8n/n8n-nodes-langchain.toolCode`  
  Python execution tool for more advanced modeling and transformations.
- **Configuration choices:**  
  - Language: Python
  - Description emphasizes forecasting, regression, scenario analysis, and financial modeling
  - No preloaded code is defined; it is available as an AI callable tool
- **Key expressions or variables used:**  
  None directly.
- **Input and output connections:**  
  Exposed as `ai_tool` to **Multi-Cloud Optimization Orchestrator**.
- **Version-specific requirements:**  
  Uses `typeVersion 1.3`.
- **Edge cases or potential failure types:**  
  - AI-generated code may fail at runtime
  - Sandboxing/environment limitations
  - Missing libraries depending on n8n execution environment
  - Long-running code or malformed input
- **Sub-workflow reference:**  
  None.

---

## 2.5 Sustainability and Executive Narrative Block

### Overview

This block equips the orchestrator with domain-specific sustainability and communication capabilities. One tool estimates carbon footprint and ESG-related improvements, while another converts technical outputs into executive-facing financial commentary.

### Nodes Involved

- Carbon Footprint Analysis Agent
- Carbon Analysis Model
- FinOps Narrative Generator Agent
- FinOps Narrative Model

### Node Details

#### Carbon Footprint Analysis Agent

- **Type and technical role:** `@n8n/n8n-nodes-langchain.agentTool`
- **Configuration choices:**  
  - Tool input text: `={{ $fromAI('resource_data', 'Resource utilization data for carbon footprint analysis') }}`
  - System message asks for CO2 estimates, regional carbon intensity, renewable energy context, migration opportunities, and ESG-aligned metrics
- **Key expressions or variables used:**  
  - `$fromAI('resource_data', 'Resource utilization data for carbon footprint analysis')`
- **Input and output connections:**  
  AI language model input from **Carbon Analysis Model**  
  Exposed as `ai_tool` to **Multi-Cloud Optimization Orchestrator**
- **Version-specific requirements:**  
  Uses `typeVersion 3`.
- **Edge cases or potential failure types:**  
  - Carbon estimates may be heuristic without a hard emissions dataset
  - Region metadata may be absent from billing export
  - Sustainability claims may be approximate and should be validated for regulated reporting
- **Sub-workflow reference:**  
  None.

#### Carbon Analysis Model

- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`
- **Configuration choices:**  
  - Model: `gpt-4o`
  - Temperature: `0.1`
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  Connected to **Carbon Footprint Analysis Agent** as `ai_languageModel`.
- **Version-specific requirements:**  
  Uses `typeVersion 1.3`.
- **Edge cases or potential failure types:**  
  Standard OpenAI API issues.
- **Sub-workflow reference:**  
  None.

#### FinOps Narrative Generator Agent

- **Type and technical role:** `@n8n/n8n-nodes-langchain.agentTool`
- **Configuration choices:**  
  - Tool input text: `={{ $fromAI('optimization_findings', 'All optimization analysis results to synthesize into narrative') }}`
  - System message instructs it to produce board-ready executive narratives, financial summaries, roadmap, risks, and ESG summary
- **Key expressions or variables used:**  
  - `$fromAI('optimization_findings', 'All optimization analysis results to synthesize into narrative')`
- **Input and output connections:**  
  AI language model input from **FinOps Narrative Model**  
  Exposed as `ai_tool` to **Multi-Cloud Optimization Orchestrator**
- **Version-specific requirements:**  
  Uses `typeVersion 3`.
- **Edge cases or potential failure types:**  
  - Narrative quality depends on the quality and completeness of upstream findings
  - May produce polished but overconfident summaries if source data is weak
- **Sub-workflow reference:**  
  None.

#### FinOps Narrative Model

- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`
- **Configuration choices:**  
  - Model: `gpt-4o`
  - Temperature: `0.3`, slightly higher to support better narrative fluency
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  Connected to **FinOps Narrative Generator Agent** as `ai_languageModel`.
- **Version-specific requirements:**  
  Uses `typeVersion 1.3`.
- **Edge cases or potential failure types:**  
  Standard OpenAI API issues.
- **Sub-workflow reference:**  
  None.

---

## 2.6 Final Report Assembly

### Overview

This block wraps the orchestrator’s structured response into a final delivery object with metadata that can be passed to future email, Slack, storage, or reporting nodes.

### Nodes Involved

- Format Final Report

### Node Details

#### Format Final Report

- **Type and technical role:** `n8n-nodes-base.set`  
  Creates a clean final JSON object for downstream use.
- **Configuration choices:**  
  Adds the following fields:
  - `report_generated_at` = current ISO timestamp
  - `report_type` = `Multi-Cloud Cost Optimization Analysis`
  - `analysis_period` = `Last 30 Days`
  - `optimization_data` = `{{ $json.output }}`
  - `status` = `Complete`
- **Key expressions or variables used:**  
  - `={{ $now.toISO() }}`
  - `={{ $json.output }}`
- **Input and output connections:**  
  Input from **Multi-Cloud Optimization Orchestrator**. No further output nodes configured.
- **Version-specific requirements:**  
  Uses `typeVersion 3.4`.
- **Edge cases or potential failure types:**  
  - If the orchestrator output does not place parsed data under `output`, `optimization_data` may be null or incorrect
  - Timestamp format depends on n8n date handling
- **Sub-workflow reference:**  
  None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Daily Cost Analysis Trigger | n8n-nodes-base.scheduleTrigger | Daily workflow entry point |  | Fetch Billing Exports | ## Fetch Billing Exports<br>**What:** Retrieves cloud billing data via HTTP GET.<br>**Why:** Centralises multi-provider cost data at source.<br><br>## How It Works<br>This workflow automates multi-cloud billing analysis and FinOps reporting using a supervised multi-agent AI architecture. It targets cloud finance teams, FinOps practitioners, DevOps leads, and CTOs seeking continuous visibility into cloud spend, resource waste, and carbon impact. A daily trigger fetches billing exports via HTTP and parses CSV data. A central Multi-Cloud Optimisation supervisor agent then coordinates four specialised sub-agents: a Resource Utilisation Analyser that identifies idle and over-provisioned assets, a Cost Optimisation Agent that surfaces savings opportunities across providers, a Carbon Footprint Analysis Agent that quantifies emissions per workload, and a FinOps Narrative Generator that produces human-readable financial commentary. Shared tools including a Financial Calculator and Advanced Analytics Code Tool support cross-agent computation. Results are parsed through a Structured Output Parser and formatted into a final consolidated report for stakeholder distribution |
| Fetch Billing Exports | n8n-nodes-base.httpRequest | Download billing export file | Daily Cost Analysis Trigger | Parse Billing Data | ## Fetch Billing Exports<br>**What:** Retrieves cloud billing data via HTTP GET.<br>**Why:** Centralises multi-provider cost data at source.<br><br>## How It Works<br>This workflow automates multi-cloud billing analysis and FinOps reporting using a supervised multi-agent AI architecture. It targets cloud finance teams, FinOps practitioners, DevOps leads, and CTOs seeking continuous visibility into cloud spend, resource waste, and carbon impact. A daily trigger fetches billing exports via HTTP and parses CSV data. A central Multi-Cloud Optimisation supervisor agent then coordinates four specialised sub-agents: a Resource Utilisation Analyser that identifies idle and over-provisioned assets, a Cost Optimisation Agent that surfaces savings opportunities across providers, a Carbon Footprint Analysis Agent that quantifies emissions per workload, and a FinOps Narrative Generator that produces human-readable financial commentary. Shared tools including a Financial Calculator and Advanced Analytics Code Tool support cross-agent computation. Results are parsed through a Structured Output Parser and formatted into a final consolidated report for stakeholder distribution |
| Parse Billing Data | n8n-nodes-base.extractFromFile | Parse billing export into structured data | Fetch Billing Exports | Multi-Cloud Optimization Orchestrator | ## Fetch Billing Exports<br>**What:** Retrieves cloud billing data via HTTP GET.<br>**Why:** Centralises multi-provider cost data at source.<br><br>## How It Works<br>This workflow automates multi-cloud billing analysis and FinOps reporting using a supervised multi-agent AI architecture. It targets cloud finance teams, FinOps practitioners, DevOps leads, and CTOs seeking continuous visibility into cloud spend, resource waste, and carbon impact. A daily trigger fetches billing exports via HTTP and parses CSV data. A central Multi-Cloud Optimisation supervisor agent then coordinates four specialised sub-agents: a Resource Utilisation Analyser that identifies idle and over-provisioned assets, a Cost Optimisation Agent that surfaces savings opportunities across providers, a Carbon Footprint Analysis Agent that quantifies emissions per workload, and a FinOps Narrative Generator that produces human-readable financial commentary. Shared tools including a Financial Calculator and Advanced Analytics Code Tool support cross-agent computation. Results are parsed through a Structured Output Parser and formatted into a final consolidated report for stakeholder distribution |
| Multi-Cloud Optimization Orchestrator | @n8n/n8n-nodes-langchain.agent | Main supervisor agent coordinating analysis | Parse Billing Data; Orchestrator Model; Resource Utilization Analyzer Agent; Cost Optimization Agent; Carbon Footprint Analysis Agent; FinOps Narrative Generator Agent; Financial Calculator; Advanced Analytics Code Tool; Structured Output Parser | Format Final Report | ## Multi-Cloud Optimisation Supervisor<br>**What:** Orchestrates four sub-agents with shared tools and memory.<br>**Why:** Coordinates parallel analysis for comprehensive coverage. |
| Orchestrator Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM for central orchestration |  | Multi-Cloud Optimization Orchestrator | ## Multi-Cloud Optimisation Supervisor<br>**What:** Orchestrates four sub-agents with shared tools and memory.<br>**Why:** Coordinates parallel analysis for comprehensive coverage.<br><br>## Setup Steps<br>1. Configure the HTTP GET node with your cloud provider.<br>2. Connect OpenAI credentials to all four sub-agent model nodes.<br>3. Link the Financial Calculator and Advanced Analytics Code Tool nodes<br>4. Configure the Structured Output Parser schema to match your reporting fields.<br>5. Test end-to-end with a sample CSV billing export before activating the daily schedule. |
| Resource Utilization Analyzer Agent | @n8n/n8n-nodes-langchain.agentTool | Tool-agent for utilization and waste detection | Utilization Analyzer Model | Multi-Cloud Optimization Orchestrator | ## Resource Utilisation Analyser Agent<br>**What:** Detects underused and idle cloud resources.<br>**Why:** Directly targets the largest source of cloud waste.<br><br>## Multi-Cloud Optimisation Supervisor<br>**What:** Orchestrates four sub-agents with shared tools and memory.<br>**Why:** Coordinates parallel analysis for comprehensive coverage. |
| Utilization Analyzer Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM for utilization analysis tool-agent |  | Resource Utilization Analyzer Agent | ## Resource Utilisation Analyser Agent<br>**What:** Detects underused and idle cloud resources.<br>**Why:** Directly targets the largest source of cloud waste. |
| Cost Optimization Agent | @n8n/n8n-nodes-langchain.agentTool | Tool-agent for financial optimization strategy | Cost Optimization Model | Multi-Cloud Optimization Orchestrator | ## Cost Optimisation, Carbon Footprint Analysis, and FinOps Narrative Generator Agents<br><br>**What:** Identify savings and rightsizing actions, quantify emissions per workload/provider, and generate financial summaries with recommendations.<br>**Why:** Reduce costs, support ESG reporting, and convert complex data into stakeholder-ready insights.<br><br>## Multi-Cloud Optimisation Supervisor<br>**What:** Orchestrates four sub-agents with shared tools and memory.<br>**Why:** Coordinates parallel analysis for comprehensive coverage. |
| Cost Optimization Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM for cost optimization tool-agent |  | Cost Optimization Agent | ## Cost Optimisation, Carbon Footprint Analysis, and FinOps Narrative Generator Agents<br><br>**What:** Identify savings and rightsizing actions, quantify emissions per workload/provider, and generate financial summaries with recommendations.<br>**Why:** Reduce costs, support ESG reporting, and convert complex data into stakeholder-ready insights. |
| Carbon Footprint Analysis Agent | @n8n/n8n-nodes-langchain.agentTool | Tool-agent for emissions and sustainability analysis | Carbon Analysis Model | Multi-Cloud Optimization Orchestrator | ## Cost Optimisation, Carbon Footprint Analysis, and FinOps Narrative Generator Agents<br><br>**What:** Identify savings and rightsizing actions, quantify emissions per workload/provider, and generate financial summaries with recommendations.<br>**Why:** Reduce costs, support ESG reporting, and convert complex data into stakeholder-ready insights.<br><br>## Multi-Cloud Optimisation Supervisor<br>**What:** Orchestrates four sub-agents with shared tools and memory.<br>**Why:** Coordinates parallel analysis for comprehensive coverage. |
| Carbon Analysis Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM for carbon analysis tool-agent |  | Carbon Footprint Analysis Agent | ## Cost Optimisation, Carbon Footprint Analysis, and FinOps Narrative Generator Agents<br><br>**What:** Identify savings and rightsizing actions, quantify emissions per workload/provider, and generate financial summaries with recommendations.<br>**Why:** Reduce costs, support ESG reporting, and convert complex data into stakeholder-ready insights. |
| FinOps Narrative Generator Agent | @n8n/n8n-nodes-langchain.agentTool | Tool-agent for executive summary and stakeholder narrative | FinOps Narrative Model | Multi-Cloud Optimization Orchestrator | ## Cost Optimisation, Carbon Footprint Analysis, and FinOps Narrative Generator Agents<br><br>**What:** Identify savings and rightsizing actions, quantify emissions per workload/provider, and generate financial summaries with recommendations.<br>**Why:** Reduce costs, support ESG reporting, and convert complex data into stakeholder-ready insights.<br><br>## Multi-Cloud Optimisation Supervisor<br>**What:** Orchestrates four sub-agents with shared tools and memory.<br>**Why:** Coordinates parallel analysis for comprehensive coverage. |
| FinOps Narrative Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM for executive narrative tool-agent |  | FinOps Narrative Generator Agent | ## Cost Optimisation, Carbon Footprint Analysis, and FinOps Narrative Generator Agents<br><br>**What:** Identify savings and rightsizing actions, quantify emissions per workload/provider, and generate financial summaries with recommendations.<br>**Why:** Reduce costs, support ESG reporting, and convert complex data into stakeholder-ready insights. |
| Financial Calculator | @n8n/n8n-nodes-langchain.toolCalculator | Arithmetic tool for exact calculations |  | Multi-Cloud Optimization Orchestrator | ## Cost Optimisation, Carbon Footprint Analysis, and FinOps Narrative Generator Agents<br><br>**What:** Identify savings and rightsizing actions, quantify emissions per workload/provider, and generate financial summaries with recommendations.<br>**Why:** Reduce costs, support ESG reporting, and convert complex data into stakeholder-ready insights. |
| Advanced Analytics Code Tool | @n8n/n8n-nodes-langchain.toolCode | Python tool for advanced modeling and forecasting |  | Multi-Cloud Optimization Orchestrator | ## Cost Optimisation, Carbon Footprint Analysis, and FinOps Narrative Generator Agents<br><br>**What:** Identify savings and rightsizing actions, quantify emissions per workload/provider, and generate financial summaries with recommendations.<br>**Why:** Reduce costs, support ESG reporting, and convert complex data into stakeholder-ready insights. |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce structured JSON output schema |  | Multi-Cloud Optimization Orchestrator |  |
| Format Final Report | n8n-nodes-base.set | Final report payload assembly | Multi-Cloud Optimization Orchestrator |  | ## Format Final Report<br>**What:** Structures all agent outputs into a unified report.<br>**Why:** Delivers a clean, consistent deliverable for distribution. |
| Sticky Note | n8n-nodes-base.stickyNote | Visual documentation |  |  |  |
| Sticky Note1 | n8n-nodes-base.stickyNote | Visual documentation |  |  |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Visual documentation |  |  |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Visual documentation |  |  |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Visual documentation |  |  |  |
| Sticky Note5 | n8n-nodes-base.stickyNote | Visual documentation |  |  |  |
| Sticky Note6 | n8n-nodes-base.stickyNote | Visual documentation |  |  |  |
| Sticky Note7 | n8n-nodes-base.stickyNote | Visual documentation |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Multi-Cloud FinOps Cost and Performance Optimization Engine`.

2. **Add a Schedule Trigger node**
   - Node type: **Schedule Trigger**
   - Name: `Daily Cost Analysis Trigger`
   - Configure it to run daily at hour `2`
   - Confirm the workflow timezone matches your reporting expectations

3. **Add an HTTP Request node**
   - Node type: **HTTP Request**
   - Name: `Fetch Billing Exports`
   - Connect it after the schedule trigger
   - Set the request URL to your billing export endpoint
   - If required, configure authentication:
     - Basic Auth
     - Header auth
     - OAuth2
     - API key
   - In response options, configure the response to be returned as a **file**
   - If you aggregate multiple clouds through one endpoint, ensure the endpoint returns a unified export file
   - If not, you may later need to expand the workflow to fetch AWS, Azure, and GCP separately and merge results

4. **Add an Extract From File node**
   - Node type: **Extract From File**
   - Name: `Parse Billing Data`
   - Connect it after `Fetch Billing Exports`
   - Leave default options unless your file format requires custom parsing
   - If your billing export is CSV, verify delimiter, headers, and encoding during testing

5. **Add the main AI agent node**
   - Node type: **AI Agent** (`@n8n/n8n-nodes-langchain.agent`)
   - Name: `Multi-Cloud Optimization Orchestrator`
   - Connect it after `Parse Billing Data`
   - Set the main text input to:
     - `={{ $json }}`
   - Enable structured output parsing
   - Set the system message to define the node as a supervisor coordinating:
     - Resource Utilization Analyzer
     - Cost Optimization Agent
     - Carbon Footprint Analyzer
     - FinOps Narrative Generator

6. **Add the orchestrator model**
   - Node type: **OpenAI Chat Model**
   - Name: `Orchestrator Model`
   - Model: `gpt-4o`
   - Temperature: `0.2`
   - Attach OpenAI credentials
   - Connect this node to the orchestrator using the **AI Language Model** connection

7. **Add the Resource Utilization tool-agent**
   - Node type: **AI Agent Tool**
   - Name: `Resource Utilization Analyzer Agent`
   - Set text input to:
     - `={{ $fromAI('billing_data', 'Billing exports and utilization logs to analyze') }}`
   - Add a system message instructing it to identify:
     - over-provisioned resources
     - underutilized clusters/instances
     - idle resources
     - right-sizing opportunities
     - workload patterns
   - Add a clear tool description
   - Connect this tool to the orchestrator using **AI Tool**

8. **Add the utilization model**
   - Node type: **OpenAI Chat Model**
   - Name: `Utilization Analyzer Model`
   - Model: `gpt-4o`
   - Temperature: `0.1`
   - Reuse the same OpenAI credentials
   - Connect it to `Resource Utilization Analyzer Agent` as **AI Language Model**

9. **Add the Cost Optimization tool-agent**
   - Node type: **AI Agent Tool**
   - Name: `Cost Optimization Agent`
   - Set text input to:
     - `={{ $fromAI('utilization_findings', 'Resource utilization analysis results to optimize') }}`
   - Configure a system message asking for:
     - 6-month predictive cost modeling
     - RI/Savings Plans recommendations
     - ROI calculations
     - commitment discount opportunities
     - carbon/cost tradeoff considerations
   - Add a tool description
   - Connect it to the orchestrator as **AI Tool**

10. **Add the cost optimization model**
    - Node type: **OpenAI Chat Model**
    - Name: `Cost Optimization Model`
    - Model: `gpt-4o`
    - Temperature: `0.2`
    - Connect it to `Cost Optimization Agent`

11. **Add the Carbon Footprint tool-agent**
    - Node type: **AI Agent Tool**
    - Name: `Carbon Footprint Analysis Agent`
    - Set text input to:
      - `={{ $fromAI('resource_data', 'Resource utilization data for carbon footprint analysis') }}`
    - Configure a system message requesting:
      - CO2 estimates by provider/region
      - renewable energy considerations
      - migration opportunities
      - sustainability improvements
      - ESG-aligned reporting metrics
    - Add a tool description
    - Connect it to the orchestrator as **AI Tool**

12. **Add the carbon model**
    - Node type: **OpenAI Chat Model**
    - Name: `Carbon Analysis Model`
    - Model: `gpt-4o`
    - Temperature: `0.1`
    - Connect it to `Carbon Footprint Analysis Agent`

13. **Add the FinOps Narrative tool-agent**
    - Node type: **AI Agent Tool**
    - Name: `FinOps Narrative Generator Agent`
    - Set text input to:
      - `={{ $fromAI('optimization_findings', 'All optimization analysis results to synthesize into narrative') }}`
    - Configure a system message instructing it to produce:
      - executive summary
      - current state assessment
      - optimization opportunities
      - risk analysis
      - six-month roadmap
      - environmental summary
    - Add a tool description
    - Connect it to the orchestrator as **AI Tool**

14. **Add the FinOps narrative model**
    - Node type: **OpenAI Chat Model**
    - Name: `FinOps Narrative Model`
    - Model: `gpt-4o`
    - Temperature: `0.3`
    - Connect it to `FinOps Narrative Generator Agent`

15. **Add a calculator tool**
    - Node type: **Calculator Tool**
    - Name: `Financial Calculator`
    - Leave default settings
    - Connect it to the orchestrator as **AI Tool**

16. **Add a code tool**
    - Node type: **Code Tool**
    - Name: `Advanced Analytics Code Tool`
    - Language: `Python`
    - Description should mention:
      - statistical analysis
      - trend forecasting
      - regression
      - compound growth
      - scenario modeling
    - Connect it to the orchestrator as **AI Tool**

17. **Add a structured output parser**
    - Node type: **Structured Output Parser**
    - Name: `Structured Output Parser`
    - Set schema type to **Manual**
    - Paste a JSON schema defining:
      - `executive_summary` as string
      - `current_state` object with cost and utilization metrics
      - `optimization_opportunities` array
      - `six_month_projection` object
      - `reserved_vs_ondemand_strategy` object
      - `carbon_footprint` object
      - `action_items` array
    - Mark at least these fields as required:
      - `executive_summary`
      - `current_state`
      - `optimization_opportunities`
      - `six_month_projection`
    - Connect it to the orchestrator as **AI Output Parser**

18. **Add a Set node for final formatting**
    - Node type: **Set**
    - Name: `Format Final Report`
    - Connect it after `Multi-Cloud Optimization Orchestrator`
    - Add these fields:
      - `report_generated_at` as string: `={{ $now.toISO() }}`
      - `report_type` as string: `Multi-Cloud Cost Optimization Analysis`
      - `analysis_period` as string: `Last 30 Days`
      - `optimization_data` as object: `={{ $json.output }}`
      - `status` as string: `Complete`

19. **Add OpenAI credentials**
    - Create or reuse an **OpenAI API credential**
    - Attach it to:
      - `Orchestrator Model`
      - `Utilization Analyzer Model`
      - `Cost Optimization Model`
      - `Carbon Analysis Model`
      - `FinOps Narrative Model`

20. **Test the billing input format**
    - Run the first three nodes:
      - Schedule Trigger manually
      - HTTP Request
      - Extract From File
    - Confirm the parsed JSON is suitable for prompt injection
    - If the dataset is too large, insert a preprocessing node before the orchestrator to:
      - aggregate spend
      - group by provider/service/region
      - reduce token volume

21. **Test the AI orchestration**
    - Execute from `Parse Billing Data` onward
    - Validate that the orchestrator:
      - calls appropriate tools
      - returns schema-compliant structured output
      - populates required numeric fields correctly

22. **Validate final output shape**
    - Inspect `Format Final Report`
    - Ensure `optimization_data` is populated
    - If the orchestrator output is not under `output`, adjust the expression accordingly

23. **Optionally add a delivery destination**
    - Although not present in the provided workflow, you can add:
      - Email node
      - Slack node
      - Google Drive / S3 / SharePoint storage node
      - Database insertion node
    - Connect it after `Format Final Report`

24. **Activate the workflow**
    - Only activate once:
      - billing endpoint works
      - AI credentials are valid
      - parser schema passes consistently
      - token size and runtime are acceptable

### Credential and Environment Expectations

- **OpenAI credential:** required on all chat model nodes
- **Billing API access:** may require internal network access or API authentication
- **Python code tool environment:** should support the code tool runtime available in your n8n deployment
- **n8n version:** this workflow uses LangChain AI nodes and modern node versions; use a recent n8n build with AI node support

### Sub-workflow Setup

This workflow does **not** contain Execute Workflow nodes or external sub-workflows. The specialist agents are internal AI tool-nodes, not separate workflows.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| ## Prerequisites - Cloud provider billing export URLs (AWS, GCP, Azure) - n8n instance (v1.0+) - HTTP access to billing APIs - Report destination (email, Slack, or storage) configured | Workflow setup guidance |
| ## Use Cases - FinOps teams generating daily multi-cloud spend digests | Workflow intent |
| ## Customisation - Add a Slack or email node to distribute the final report automatically | Extension idea |
| ## Benefits - Daily automation eliminates manual billing export and analysis effort | Operational value |
| ## Setup Steps 1. Configure the HTTP GET node with your cloud provider. 2. Connect OpenAI credentials to all four sub-agent model nodes. 3. Link the Financial Calculator and Advanced Analytics Code Tool nodes. 4. Configure the Structured Output Parser schema to match your reporting fields. 5. Test end-to-end with a sample CSV billing export before activating the daily schedule. | Build and validation guidance |
| ## How It Works This workflow automates multi-cloud billing analysis and FinOps reporting using a supervised multi-agent AI architecture. It targets cloud finance teams, FinOps practitioners, DevOps leads, and CTOs seeking continuous visibility into cloud spend, resource waste, and carbon impact. A daily trigger fetches billing exports via HTTP and parses CSV data. A central Multi-Cloud Optimisation supervisor agent then coordinates four specialised sub-agents: a Resource Utilisation Analyser that identifies idle and over-provisioned assets, a Cost Optimisation Agent that surfaces savings opportunities across providers, a Carbon Footprint Analysis Agent that quantifies emissions per workload, and a FinOps Narrative Generator that produces human-readable financial commentary. Shared tools including a Financial Calculator and Advanced Analytics Code Tool support cross-agent computation. Results are parsed through a Structured Output Parser and formatted into a final consolidated report for stakeholder distribution. | Overall workflow explanation |

## Additional implementation note

The current workflow retrieves data from a single billing export endpoint placeholder. If your Azure, AWS, and GCP billing data are exposed through separate APIs or files, you will need to extend the ingestion block with multiple HTTP Request nodes plus normalization/merge logic before passing data to the orchestrator.
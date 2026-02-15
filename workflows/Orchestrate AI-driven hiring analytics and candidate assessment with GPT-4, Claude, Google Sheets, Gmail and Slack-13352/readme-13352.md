Orchestrate AI-driven hiring analytics and candidate assessment with GPT-4, Claude, Google Sheets, Gmail and Slack

https://n8nworkflows.xyz/workflows/orchestrate-ai-driven-hiring-analytics-and-candidate-assessment-with-gpt-4--claude--google-sheets--gmail-and-slack-13352


# Orchestrate AI-driven hiring analytics and candidate assessment with GPT-4, Claude, Google Sheets, Gmail and Slack

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name (in JSON):** *AI-driven interview scheduling & multi-model candidate assessment system*  
**User-provided title:** *Orchestrate AI-driven hiring analytics and candidate assessment with GPT-4, Claude, Google Sheets, Gmail and Slack*

### Purpose
This workflow runs **daily hiring funnel analytics**, uses **GPT-4-class models** to generate structured insights, then an **orchestration agent** decides which specialized analysis tools to invoke (sourcing, interview process, offer/assessment). Results are **routed by priority**, stored in **n8n Data Tables**, and for critical/high priority cases, **Slack and email notifications** are sent before all results are consolidated and archived.

### Notable alignment gaps vs title/notes
- **Claude**: not present in the JSON (all LMs are OpenAI).
- **Google Sheets**: not present; storage is via **n8n Data Table** nodes.
- **Gmail**: not present as a Gmail node; email is sent via **Email Send** (SMTP-style) node.

### Logical Blocks
**1.1 Scheduled trigger & configuration**  
Runs daily, sets environment-like configuration (channel/email/goals).

**1.2 Metric preparation**  
Builds a metrics payload (currently hard-coded sample values).

**1.3 Funnel analytics (LLM + structured parsing)**  
GPT model produces a structured funnel health report.

**1.4 Multi-agent orchestration (LLM agent + tool calls)**  
Orchestration agent decides whether to call specialized tools (sourcing/interview/assessment) and returns consolidated actions.

**1.5 Priority routing & persistence**  
Routes by overall priority, stores into separate Data Tables; triggers alerts for critical/high.

**1.6 Notifications, consolidation & archival**  
Sends Slack + email for critical/high; merges results from all paths and archives a full report.

---

## 2. Block-by-Block Analysis

### 2.1 Scheduled trigger & configuration

**Overview:** Initiates the workflow daily and defines key configuration values (Slack channel, leadership email, and hiring goals) used by downstream nodes.

**Nodes Involved:**
- Daily Hiring Analytics Trigger
- Workflow Configuration

#### Node: Daily Hiring Analytics Trigger
- **Type / role:** `Schedule Trigger` — entry point; executes on schedule.
- **Configuration:** Runs daily at **09:00** (based on n8n instance timezone).
- **Outputs:** One execution item to **Workflow Configuration**.
- **Edge cases / failures:**
  - Timezone mismatch if instance timezone differs from intended HR timezone.
  - If workflow is inactive (it is `active: false`), schedule will not run.

#### Node: Workflow Configuration
- **Type / role:** `Set` — creates configuration variables reused via expressions.
- **Key fields set:**
  - `slackHRChannel` (placeholder)
  - `leadershipEmail` (placeholder)
  - `criticalThreshold` = 80 (note: not actually used downstream)
  - Hiring goals: `hiringGoalApplications`=100, `hiringGoalInterviews`=30, `hiringGoalOffers`=10
- **Expressions used by others:**
  - `$('Workflow Configuration').first().json.slackHRChannel`
  - `$('Workflow Configuration').first().json.leadershipEmail`
  - goal values referenced in funnel prompt
- **Connections:** Outputs to **Prepare Hiring Metrics Data**.
- **Edge cases / failures:**
  - Missing/invalid Slack channel ID causes Slack node failure.
  - Invalid leadership email breaks email delivery.
  - `criticalThreshold` is defined but unused → potential logic bug / incomplete feature.

**Sticky note context (applies to this area):**
- **Sticky Note2 – “How It Works”**: describes end-to-end recruitment automation with multi-model assessment and consolidation/notifications (conceptually; implementation differs: no Google Sheets, no Claude).
- **Sticky Note1 – “Setup Steps”**: lists required integrations (some not present in JSON).

---

### 2.2 Metric preparation

**Overview:** Builds the hiring funnel metrics payload for analysis. Currently uses fixed numbers and arrays (example data), plus a dynamic report timestamp.

**Nodes Involved:**
- Prepare Hiring Metrics Data

#### Node: Prepare Hiring Metrics Data
- **Type / role:** `Set` — constructs the metrics object used by the analytics agent.
- **Key fields:**
  - Volumes: `totalApplications`=150, `screenedCandidates`=85, `phoneInterviews`=42, `onsiteInterviews`=28
  - Offers: `offersMade`=12, `offersAccepted`=9
  - `averageTimeToHire`=35 (days)
  - Arrays:
    - `topSourceChannels` = ["LinkedIn","Referrals","Indeed","Company Website"]
    - `openPositions` = [{role,count}, ...]
  - `reportDate` = `{{ $now.toISO() }}`
- **Connections:** Outputs to **Funnel Analytics Agent**.
- **Edge cases / failures:**
  - Data is hard-coded; if intended to be real analytics, you’ll need upstream data ingestion (ATS/Sheets/DB).
  - If numeric fields are missing/0, LLM calculations may produce divide-by-zero or misleading rates unless guarded in prompt.

**Sticky note context:**
- **Sticky Note3 – “Analytics Framework Setup”** (conceptual): mentions “Former Analytics Agent”; in this workflow the equivalent is **Funnel Analytics Agent + Funnel Analytics Output Parser**.

---

### 2.3 Funnel analytics (LLM + structured parsing)

**Overview:** Uses an LLM agent to analyze funnel conversion, goals, bottlenecks, sources, and returns a strict structured JSON-like object enforced by a schema parser.

**Nodes Involved:**
- Funnel Analytics Agent
- OpenAI Model - Funnel Analytics
- Funnel Analytics Output Parser

#### Node: OpenAI Model - Funnel Analytics
- **Type / role:** `LM Chat OpenAI` — language model backend for the analytics agent.
- **Model:** `gpt-4o`
- **Temperature:** 0.2 (more deterministic/analytical).
- **Credential:** OpenAI API credential “OpenAi account”.
- **Connections:** Provides `ai_languageModel` input to **Funnel Analytics Agent**.
- **Edge cases / failures:**
  - OpenAI auth / quota / model access errors.
  - Transient timeouts; consider retries at node level if supported.

#### Node: Funnel Analytics Output Parser
- **Type / role:** `Structured Output Parser` — enforces schema on agent output.
- **Schema highlights (required):**
  - `overallHealth` (Excellent/Good/Fair/Poor/Critical)
  - `conversionRates` with numeric stage conversions
  - `bottlenecks` array of {stage, severity, impact}
  - `goalPerformance` ratios vs goals
  - `topRecommendations` array
  - `priority` (High/Medium/Low)
  - `reasoning` string
- **Connections:** Feeds `ai_outputParser` into **Funnel Analytics Agent**.
- **Edge cases / failures:**
  - If the model output can’t be coerced into schema, parser fails and blocks downstream steps.
  - Numeric types: the model may output percentages as strings (“45%”)—parser expects numbers.

#### Node: Funnel Analytics Agent
- **Type / role:** `LangChain Agent` — runs the prompt, using model + output parser.
- **Prompt content:** Injects metrics and goals via expressions, e.g.:
  - `{{ $json.totalApplications }}` etc.
  - `{{ $('Workflow Configuration').first().json.hiringGoalApplications }}`
- **System message:** “Hiring Funnel Analytics Specialist” with explicit step list (conversion rates, bottlenecks, goals, sources, time-to-hire, acceptance, recommendations).
- **Connections:** Main output to **Orchestration Agent**.
- **Outputs:** Produces `output` object (as typical with structured parser) consumed later by orchestration prompt.
- **Edge cases / failures:**
  - If upstream metrics missing, prompt yields incomplete analysis; consider validation node.
  - If goal fields absent, expressions referencing Workflow Configuration can fail (if node not executed or empty items).

---

### 2.4 Multi-agent orchestration (agent + tool calls)

**Overview:** A second agent consumes funnel analysis and dynamically calls specialized “agent tools” (sourcing, interview optimization, assessment/offer strategy). It then outputs a structured action plan.

**Nodes Involved:**
- Orchestration Agent
- OpenAI Model - Orchestration
- Orchestration Output Parser
- Candidate Sourcing Agent Tool (+ OpenAI Model - Sourcing + Sourcing Output Parser)
- Interview Scheduling Agent Tool (+ OpenAI Model - Interview + Interview Output Parser)
- Candidate Assessment Agent Tool (+ OpenAI Model - Assessment + Assessment Output Parser)

#### Node: OpenAI Model - Orchestration
- **Type / role:** `LM Chat OpenAI`
- **Model:** `gpt-4o`
- **Temperature:** 0.3
- **Connections:** Supplies model to **Orchestration Agent**.
- **Edge cases:** same OpenAI risks (auth/quota/timeouts).

#### Node: Orchestration Output Parser
- **Type / role:** `Structured Output Parser`
- **Schema highlights:**
  - Required: `orchestrationSummary`, `agentsCalled`, `consolidatedActions`, `overallPriority`
  - Optional: `sourcingInsights`, `interviewInsights`, `assessmentInsights`, `criticalAlerts`
- **Connections:** Supplies output parser to **Orchestration Agent**.
- **Edge cases:**
  - If `overallPriority` is missing or different (e.g., “HIGH”), downstream Switch will route to fallback.

#### Node: Candidate Sourcing Agent Tool
- **Type / role:** `Agent Tool` — callable tool for the orchestration agent.
- **Input mapping:** `{{ $fromAI("funnelData", "...", "json") }}`  
  This expects the orchestrator to pass a JSON object under the key `funnelData`.
- **System message:** Sourcing specialist focusing on channel performance and actions per open position.
- **Tool description:** “Analyzes sourcing channel effectiveness…”
- **Dependencies / connections:**
  - Receives `ai_languageModel` from **OpenAI Model - Sourcing**
  - Receives `ai_outputParser` from **Sourcing Output Parser**
  - Returns to **Orchestration Agent** via `ai_tool` connection
- **Edge cases:**
  - If orchestrator doesn’t provide valid JSON tool input, `$fromAI` may fail or pass null-like content.
  - Tool outputs must satisfy its parser schema.

#### Node: OpenAI Model - Sourcing
- **Type / role:** `LM Chat OpenAI`
- **Model:** `gpt-4o-mini`
- **Temperature:** 0.2
- **Connections:** Provides model to Candidate Sourcing Agent Tool.

#### Node: Sourcing Output Parser
- **Type / role:** `Structured Output Parser`
- **Required:**
  - `channelPerformance` array
  - `sourcingActions` array
  - `estimatedApplicationIncrease` number
- **Connections:** Provides parser to Candidate Sourcing Agent Tool.
- **Edge cases:** Percent outputs must be numeric.

#### Node: Interview Scheduling Agent Tool
- **Type / role:** `Agent Tool` — callable tool for interview process optimization.
- **Input mapping:** `{{ $fromAI("interviewData", "...", "json") }}`
- **Dependencies / connections:**
  - Model: **OpenAI Model - Interview**
  - Parser: **Interview Output Parser**
  - Returns tool output to **Orchestration Agent**
- **Edge cases:** Missing/invalid JSON tool args; schema mismatch.

#### Node: OpenAI Model - Interview
- **Type / role:** `LM Chat OpenAI`
- **Model:** `gpt-4o-mini`
- **Temperature:** 0.2

#### Node: Interview Output Parser
- **Type / role:** `Structured Output Parser`
- **Required:**
  - `processImprovements` array
  - `schedulingOptimizations` array
  - `estimatedConversionIncrease` number
- **Optional:** `candidateExperienceEnhancements`, `estimatedTimeReduction`
- **Edge cases:** Ensure numeric outputs are numbers.

#### Node: Candidate Assessment Agent Tool
- **Type / role:** `Agent Tool` — callable tool for offer acceptance & quality-of-hire strategy.
- **Input mapping:** `{{ $fromAI("offerData", "...", "json") }}`
- **Dependencies / connections:**
  - Model: **OpenAI Model - Assessment**
  - Parser: **Assessment Output Parser**
  - Returns to **Orchestration Agent**
- **Edge cases:** Same tool input + schema enforcement risks.

#### Node: OpenAI Model - Assessment
- **Type / role:** `LM Chat OpenAI`
- **Model:** `gpt-4o-mini`
- **Temperature:** 0.2

#### Node: Assessment Output Parser
- **Type / role:** `Structured Output Parser`
- **Required:**
  - `offerStrategies` array
  - `qualityImprovements` array
  - `estimatedAcceptanceIncrease` number
  - `competitivenessScore` number (0–100 intended)
- **Edge cases:** Model may output score as string; parser expects number.

#### Node: Orchestration Agent
- **Type / role:** `LangChain Agent` — orchestrates tool usage and final plan.
- **Prompt:** Uses funnel analysis output:
  - `Based on the funnel analysis: {{ JSON.stringify($json.output) }}, coordinate hiring workflow actions...`
- **System message:** Explicit instructions to call tools based on bottlenecks and return structured results.
- **Connections:** Outputs to **Route by Priority**.
- **Edge cases:**
  - If funnel analysis structure differs (parser failure upstream), orchestration input will be missing.
  - Tool call selection is probabilistic; even with instructions, the agent may under/over-call tools unless strongly constrained.

**Sticky note context:**
- **Sticky Note5 – “Intelligent Routing & AI Assessment”**: matches this block’s intent (agent distributes work to specialist modules).

---

### 2.5 Priority routing & persistence

**Overview:** Routes orchestration output by `overallPriority` and stores insights into separate Data Tables. High priority additionally triggers a criticality check for alerts.

**Nodes Involved:**
- Route by Priority
- Store High Priority Insights
- Store Medium Priority Insights
- Store Low Priority Insights
- Check Critical Threshold

#### Node: Route by Priority
- **Type / role:** `Switch` — routes items into High/Medium/Low outputs.
- **Rules:** Compares `{{ $json.output.overallPriority }}` to `High`, `Medium`, `Low`.
- **Fallback:** “Default” output if none match.
- **Connections:**
  - High → Store High Priority Insights
  - Medium → Store Medium Priority Insights
  - Low → Store Low Priority Insights
- **Edge cases:**
  - If `overallPriority` missing or different casing, item goes to fallback (but fallback is not connected → data loss).
  - If orchestration output parser uses different field (e.g., `priority`), routing fails.

#### Node: Store High Priority Insights
- **Type / role:** `Data Table`
- **Target table:** `HighPriorityHiringInsights`
- **Connections:** Outputs to **Check Critical Threshold**
- **Edge cases:**
  - Data Table not created yet / permission issues.
  - Schema drift: Data Tables store JSON-like rows; ensure expected columns/fields strategy.

#### Node: Store Medium Priority Insights
- **Type / role:** `Data Table`
- **Target table:** `MediumPriorityHiringInsights`
- **Connections:** Outputs to **Consolidate All Insights** (input 0).
- **Edge cases:** same as above.

#### Node: Store Low Priority Insights
- **Type / role:** `Data Table`
- **Target table:** `LowPriorityHiringInsights`
- **Connections:** Outputs to **Consolidate All Insights** (input 1).
- **Edge cases:** same as above.

#### Node: Check Critical Threshold
- **Type / role:** `IF`
- **Condition (OR):**
  - `criticalAlerts` is **not empty** (`{{ $json.output.criticalAlerts }}` array notEmpty), OR
  - `overallPriority == "High"`
- **Connections:** True path triggers:
  - **Notify HR Team - High Priority**
  - **Email Leadership - Critical Insights**
  (No false path connected.)
- **Edge cases / failures:**
  - If `criticalAlerts` is undefined, array `notEmpty` may behave as false (ok) but depends on n8n evaluation; safest is to default to `[]`.
  - This node does **not** use `criticalThreshold` from configuration—threshold logic is effectively “alerts exist OR high priority”.

**Sticky note context:**
- **Sticky Note4 – “Data Consolidation & Notification”**: conceptually aligns, though storage is Data Tables (not Google Sheets).

---

### 2.6 Notifications, consolidation & archival

**Overview:** Sends Slack and email alerts for critical/high cases, then merges outputs from multiple paths and archives a combined report.

**Nodes Involved:**
- Notify HR Team - High Priority
- Email Leadership - Critical Insights
- Consolidate All Insights
- Archive Complete Analytics Report

#### Node: Notify HR Team - High Priority
- **Type / role:** `Slack`
- **Auth:** OAuth2 (Slack credential “Slack account”)
- **Channel selection:** `channelId` from configuration:
  - `={{ $('Workflow Configuration').first().json.slackHRChannel }}`
- **Message:** Formatted text includes:
  - Orchestration summary
  - Critical alerts (bulleted if present)
  - Agents called
  - Top 5 actions (priority, action, owner)
  - Timestamp: `{{ $now.toFormat("yyyy-MM-dd HH:mm") }}`
- **Connections:** Outputs to **Consolidate All Insights** (input 0).
- **Edge cases:**
  - Invalid channel ID / missing Slack scopes.
  - Slack rate limits for frequent posts (less likely for daily schedule).
  - Uses emoji and markdown; if Slack formatting changes, readability may degrade.

#### Node: Email Leadership - Critical Insights
- **Type / role:** `Email Send`
- **To:** `={{ $('Workflow Configuration').first().json.leadershipEmail }}`
- **From:** placeholder sender email (must be valid for SMTP relay)
- **Subject:** `CRITICAL: Hiring Analytics Alert - YYYY-MM-DD`
- **Body:** Rich HTML including orchestration summary, critical alerts, agents engaged, and a table of up to 10 actions.
- **Connections:** Outputs to **Consolidate All Insights** (input 1).
- **Edge cases:**
  - Missing SMTP credentials/config in n8n (not shown in JSON; node requires email server setup).
  - Sender domain policies (SPF/DKIM/DMARC) causing delivery issues.
  - If `criticalAlerts` undefined, the HTML expression handles it with a fallback “None”.

#### Node: Consolidate All Insights
- **Type / role:** `Merge`
- **Mode:** `combine` / `combineByPosition`
- **Inputs connected:**
  - From **Notify HR Team - High Priority** (index 0)
  - From **Store Medium Priority Insights** (index 0)
  - From **Store Low Priority Insights** (index 1)
  - From **Email Leadership - Critical Insights** (index 1)
- **Important behavior note:** `combineByPosition` expects aligned item positions across inputs. If only one branch executes (common), merge output can be empty or partially combined depending on how many inputs provide items.
- **Connections:** Outputs to **Archive Complete Analytics Report**
- **Edge cases / failures:**
  - If Medium/Low paths produce items but High alert path doesn’t (or vice versa), the merge may not behave as intended.
  - Consider switching to a merge strategy that appends or merges by key to avoid positional dependency.

#### Node: Archive Complete Analytics Report
- **Type / role:** `Data Table`
- **Target table:** `CompleteHiringAnalyticsArchive`
- **Purpose:** Stores the final consolidated report for auditing/tracking.
- **Edge cases:** Data Table availability/permissions; output size if LLM reasoning is long.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Daily Hiring Analytics Trigger | Schedule Trigger | Daily entry point | — | Workflow Configuration | ## How It Works (conceptual end-to-end pipeline; mentions Sheets/Claude not present) |
| Workflow Configuration | Set | Central config (channels, emails, goals) | Daily Hiring Analytics Trigger | Prepare Hiring Metrics Data | ## Setup Steps (mentions OpenAI/Claude/Sheets/Gmail/Slack; some not present) |
| Prepare Hiring Metrics Data | Set | Builds funnel metrics payload | Workflow Configuration | Funnel Analytics Agent | ## Analytics Framework Setup (structured evaluation criteria via agent+parser) |
| OpenAI Model - Funnel Analytics | OpenAI Chat Model | LLM backend for funnel analysis | — | Funnel Analytics Agent | ## Analytics Framework Setup (structured evaluation criteria via agent+parser) |
| Funnel Analytics Output Parser | Structured Output Parser | Enforces schema for funnel analysis | — | Funnel Analytics Agent | ## Analytics Framework Setup (structured evaluation criteria via agent+parser) |
| Funnel Analytics Agent | LangChain Agent | Computes funnel health/bottlenecks/goals | Prepare Hiring Metrics Data | Orchestration Agent | ## Analytics Framework Setup (structured evaluation criteria via agent+parser) |
| OpenAI Model - Orchestration | OpenAI Chat Model | LLM backend for orchestration agent | — | Orchestration Agent | ## Intelligent Routing & AI Assessment (agent distributes to specialized evaluators) |
| Orchestration Output Parser | Structured Output Parser | Enforces schema for orchestration plan | — | Orchestration Agent | ## Intelligent Routing & AI Assessment (agent distributes to specialized evaluators) |
| Candidate Sourcing Agent Tool | Agent Tool | Tool: sourcing channel optimization | OpenAI Model - Sourcing; Sourcing Output Parser | Orchestration Agent | ## Intelligent Routing & AI Assessment (agent distributes to specialized evaluators) |
| OpenAI Model - Sourcing | OpenAI Chat Model | LLM backend for sourcing tool | — | Candidate Sourcing Agent Tool | ## Intelligent Routing & AI Assessment (agent distributes to specialized evaluators) |
| Sourcing Output Parser | Structured Output Parser | Enforces schema for sourcing tool output | — | Candidate Sourcing Agent Tool | ## Intelligent Routing & AI Assessment (agent distributes to specialized evaluators) |
| Interview Scheduling Agent Tool | Agent Tool | Tool: interview process optimization | OpenAI Model - Interview; Interview Output Parser | Orchestration Agent | ## Intelligent Routing & AI Assessment (agent distributes to specialized evaluators) |
| OpenAI Model - Interview | OpenAI Chat Model | LLM backend for interview tool | — | Interview Scheduling Agent Tool | ## Intelligent Routing & AI Assessment (agent distributes to specialized evaluators) |
| Interview Output Parser | Structured Output Parser | Enforces schema for interview tool output | — | Interview Scheduling Agent Tool | ## Intelligent Routing & AI Assessment (agent distributes to specialized evaluators) |
| Candidate Assessment Agent Tool | Agent Tool | Tool: offer/assessment strategy | OpenAI Model - Assessment; Assessment Output Parser | Orchestration Agent | ## Intelligent Routing & AI Assessment (agent distributes to specialized evaluators) |
| OpenAI Model - Assessment | OpenAI Chat Model | LLM backend for assessment tool | — | Candidate Assessment Agent Tool | ## Intelligent Routing & AI Assessment (agent distributes to specialized evaluators) |
| Assessment Output Parser | Structured Output Parser | Enforces schema for assessment tool output | — | Candidate Assessment Agent Tool | ## Intelligent Routing & AI Assessment (agent distributes to specialized evaluators) |
| Orchestration Agent | LangChain Agent | Decides tool calls + unified action plan | Funnel Analytics Agent | Route by Priority | ## Intelligent Routing & AI Assessment (agent distributes to specialized evaluators) |
| Route by Priority | Switch | Routes by overallPriority | Orchestration Agent | Store High Priority Insights; Store Medium Priority Insights; Store Low Priority Insights | ## Data Consolidation & Notification (aggregation + alerts; mentions Sheets/Gmail conceptually) |
| Store High Priority Insights | Data Table | Persist high-priority results | Route by Priority (High) | Check Critical Threshold | ## Data Consolidation & Notification (aggregation + alerts; mentions Sheets/Gmail conceptually) |
| Check Critical Threshold | IF | Determines if alerts should be sent | Store High Priority Insights | Notify HR Team - High Priority; Email Leadership - Critical Insights | ## Data Consolidation & Notification (aggregation + alerts; mentions Sheets/Gmail conceptually) |
| Notify HR Team - High Priority | Slack | Slack alert to HR channel | Check Critical Threshold (true) | Consolidate All Insights | ## Data Consolidation & Notification (aggregation + alerts; mentions Sheets/Gmail conceptually) |
| Email Leadership - Critical Insights | Email Send | Email critical report to leadership | Check Critical Threshold (true) | Consolidate All Insights | ## Data Consolidation & Notification (aggregation + alerts; mentions Sheets/Gmail conceptually) |
| Store Medium Priority Insights | Data Table | Persist medium-priority results | Route by Priority (Medium) | Consolidate All Insights | ## Data Consolidation & Notification (aggregation + alerts; mentions Sheets/Gmail conceptually) |
| Store Low Priority Insights | Data Table | Persist low-priority results | Route by Priority (Low) | Consolidate All Insights | ## Data Consolidation & Notification (aggregation + alerts; mentions Sheets/Gmail conceptually) |
| Consolidate All Insights | Merge | Combines outputs from branches | Notify HR Team - High Priority; Store Medium Priority Insights; Store Low Priority Insights; Email Leadership - Critical Insights | Archive Complete Analytics Report | ## Data Consolidation & Notification (aggregation + alerts; mentions Sheets/Gmail conceptually) |
| Archive Complete Analytics Report | Data Table | Archive consolidated report | Consolidate All Insights | — | ## Data Consolidation & Notification (aggregation + alerts; mentions Sheets/Gmail conceptually) |
| Sticky Note | Sticky Note | Documentation: prerequisites/use cases/customization/benefits | — | — | ## Prerequisites / Use Cases / Customization / Benefits |
| Sticky Note1 | Sticky Note | Documentation: setup steps | — | — | ## Setup Steps |
| Sticky Note2 | Sticky Note | Documentation: how it works | — | — | ## How It Works |
| Sticky Note3 | Sticky Note | Documentation: analytics framework | — | — | ## Analytics Framework Setup |
| Sticky Note4 | Sticky Note | Documentation: consolidation & notification | — | — | ## Data Consolidation & Notification |
| Sticky Note5 | Sticky Note | Documentation: intelligent routing | — | — | ## Intelligent Routing & AI Assessment |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it (e.g.) “AI-driven interview scheduling & multi-model candidate assessment system”.
   - Keep it **inactive** until credentials and tables are ready.

2. **Add trigger**
   - Add **Schedule Trigger** node.
   - Configure: run daily at **09:00**.

3. **Add configuration node**
   - Add **Set** node named **Workflow Configuration**.
   - Add fields:
     - `slackHRChannel` (string): Slack channel ID (e.g., `C0123...`)
     - `leadershipEmail` (string): distribution email
     - `criticalThreshold` (number): 80 (optional; currently not used later)
     - `hiringGoalApplications` (number): 100
     - `hiringGoalInterviews` (number): 30
     - `hiringGoalOffers` (number): 10
   - Connect: **Schedule Trigger → Workflow Configuration**

4. **Add metrics preparation**
   - Add **Set** node named **Prepare Hiring Metrics Data**.
   - Add numeric metrics fields (or replace with real data sources later):
     - `totalApplications`, `screenedCandidates`, `phoneInterviews`, `onsiteInterviews`, `offersMade`, `offersAccepted`, `averageTimeToHire`
   - Add arrays:
     - `topSourceChannels` (array of strings)
     - `openPositions` (array of objects with `role`, `count`)
   - Add `reportDate` (string) as expression: `{{ $now.toISO() }}`
   - Connect: **Workflow Configuration → Prepare Hiring Metrics Data**

5. **Add Funnel Analytics LLM components**
   - Add **LM Chat OpenAI** node named **OpenAI Model - Funnel Analytics**
     - Model: `gpt-4o`
     - Temperature: `0.2`
     - Set OpenAI credential (API key with access to chosen model).
   - Add **Structured Output Parser** named **Funnel Analytics Output Parser**
     - Schema: create the object schema with required fields:
       `overallHealth`, `conversionRates`, `bottlenecks`, `goalPerformance`, `topRecommendations`, `priority`, `reasoning`.
   - Add **LangChain Agent** named **Funnel Analytics Agent**
     - Prompt: include the metrics and goals via expressions from current item and the configuration node.
     - System message: funnel analytics specialist instructions.
     - Enable output parsing and attach **Funnel Analytics Output Parser**.
     - Attach model **OpenAI Model - Funnel Analytics**.
   - Connect: **Prepare Hiring Metrics Data → Funnel Analytics Agent**

6. **Add Orchestration LLM components**
   - Add **LM Chat OpenAI** node named **OpenAI Model - Orchestration**
     - Model: `gpt-4o`
     - Temperature: `0.3`
   - Add **Structured Output Parser** named **Orchestration Output Parser**
     - Schema requires: `orchestrationSummary`, `agentsCalled`, `consolidatedActions`, `overallPriority`.
     - Optional: `criticalAlerts`, and per-tool insight objects.
   - Add **LangChain Agent** named **Orchestration Agent**
     - Prompt: reference funnel analysis as `{{ JSON.stringify($json.output) }}`.
     - System message: instruct when to call each tool and to synthesize outputs.
     - Attach **OpenAI Model - Orchestration** and **Orchestration Output Parser**.
   - Connect: **Funnel Analytics Agent → Orchestration Agent**

7. **Create specialized agent tools (3 tools)**
   - For each tool, create:
     1) **LM Chat OpenAI** (gpt-4o-mini, temperature 0.2)  
     2) **Structured Output Parser** with the tool’s schema  
     3) **Agent Tool** node with:
        - Tool description
        - System message for the specialist
        - Input mapping using `$fromAI(...)` expecting JSON arguments

   **7A. Sourcing tool**
   - Nodes:
     - **OpenAI Model - Sourcing** (`gpt-4o-mini`)
     - **Sourcing Output Parser** (requires `channelPerformance`, `sourcingActions`, `estimatedApplicationIncrease`)
     - **Candidate Sourcing Agent Tool**
       - Text: `{{ $fromAI("funnelData", "...", "json") }}`
   - Connect:
     - Model → Tool via `ai_languageModel`
     - Parser → Tool via `ai_outputParser`
     - Tool → **Orchestration Agent** via `ai_tool`

   **7B. Interview tool**
   - Nodes:
     - **OpenAI Model - Interview**
     - **Interview Output Parser** (requires `processImprovements`, `schedulingOptimizations`, `estimatedConversionIncrease`)
     - **Interview Scheduling Agent Tool**
       - Text: `{{ $fromAI("interviewData", "...", "json") }}`
   - Connect similarly to orchestration agent via `ai_tool`.

   **7C. Assessment tool**
   - Nodes:
     - **OpenAI Model - Assessment**
     - **Assessment Output Parser** (requires `offerStrategies`, `qualityImprovements`, `estimatedAcceptanceIncrease`, `competitivenessScore`)
     - **Candidate Assessment Agent Tool**
       - Text: `{{ $fromAI("offerData", "...", "json") }}`
   - Connect similarly to orchestration agent via `ai_tool`.

8. **Add priority routing**
   - Add **Switch** node named **Route by Priority**.
   - Add three rules checking expression `{{ $json.output.overallPriority }}` equals `High`, `Medium`, `Low`.
   - (Recommended) Connect fallback to a “Default storage” node to avoid dropping data.
   - Connect: **Orchestration Agent → Route by Priority**

9. **Add Data Tables for storage**
   - Add three **Data Table** nodes:
     - **Store High Priority Insights** → table `HighPriorityHiringInsights`
     - **Store Medium Priority Insights** → table `MediumPriorityHiringInsights`
     - **Store Low Priority Insights** → table `LowPriorityHiringInsights`
   - Create these Data Tables in n8n (or allow node to create, depending on your environment).
   - Connect:
     - Route High → Store High
     - Route Medium → Store Medium
     - Route Low → Store Low

10. **Add critical/high alert gating**
    - Add **IF** node named **Check Critical Threshold**:
      - OR condition:
        - `{{ $json.output.criticalAlerts }}` is not empty
        - OR `{{ $json.output.overallPriority }}` equals `High`
    - Connect: **Store High Priority Insights → Check Critical Threshold**

11. **Add notifications**
    - Add **Slack** node named **Notify HR Team - High Priority**
      - Auth: Slack OAuth2 credential
      - Channel ID: `{{ $('Workflow Configuration').first().json.slackHRChannel }}`
      - Message: include orchestration summary, critical alerts, agents called, top actions.
    - Add **Email Send** node named **Email Leadership - Critical Insights**
      - To: `{{ $('Workflow Configuration').first().json.leadershipEmail }}`
      - From: a valid sender address for your SMTP setup
      - Subject/body: HTML with key insights.
    - Connect IF true output to both Slack and Email nodes.

12. **Consolidate and archive**
    - Add **Merge** node named **Consolidate All Insights**
      - Mode: `combine` → `combineByPosition` (as in JSON)
      - Connect outputs into merge (note this is brittle; consider append/aggregate alternative in production):
        - Slack → Merge input 0
        - Medium store → Merge input 0
        - Low store → Merge input 1
        - Email → Merge input 1
    - Add **Data Table** node named **Archive Complete Analytics Report**
      - Table: `CompleteHiringAnalyticsArchive`
    - Connect: **Merge → Archive**

13. **Credentials checklist**
    - **OpenAI**: API key with access to `gpt-4o` and `gpt-4o-mini`.
    - **Slack OAuth2**: workspace authorization + scopes to post in target channel.
    - **Email Send**: configure SMTP credentials in n8n (node-level or global, depending on deployment).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Prerequisites: Developer account with API access, OpenAI API key (GPT-4 enabled)… Benefits: Reduces time-to-hire by 60% through parallel AI assessments.” | Sticky Note (workflow documentation) |
| “Setup Steps… Link Anthropic Claude API… Authorize Google Sheets… Set up Gmail…” | Sticky Note1 (conceptual; Claude/Sheets/Gmail are not implemented in this JSON) |
| “How It Works… consolidated insights automatically populate Google Sheets… Gmail notifications… multi-model (OpenAI GPT-4, Claude)” | Sticky Note2 (conceptual; actual implementation uses Data Tables + Email Send + OpenAI only) |
| “Analytics Framework Setup… structured evaluation criteria…” | Sticky Note3 |
| “Data Consolidation & Notification… centralized tracking + alerts…” | Sticky Note4 |
| “Intelligent Routing & AI Assessment… orchestration agent distributes to specialized evaluators…” | Sticky Note5 |
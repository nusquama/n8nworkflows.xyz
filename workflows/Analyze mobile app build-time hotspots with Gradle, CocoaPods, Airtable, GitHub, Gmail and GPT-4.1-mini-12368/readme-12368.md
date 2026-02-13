Analyze mobile app build-time hotspots with Gradle, CocoaPods, Airtable, GitHub, Gmail and GPT-4.1-mini

https://n8nworkflows.xyz/workflows/analyze-mobile-app-build-time-hotspots-with-gradle--cocoapods--airtable--github--gmail-and-gpt-4-1-mini-12368


# Analyze mobile app build-time hotspots with Gradle, CocoaPods, Airtable, GitHub, Gmail and GPT-4.1-mini

## 1. Workflow Overview

**Purpose:** This workflow acts as an automated build-performance monitor for mobile apps (Android via **Gradle** and iOS via **CocoaPods**) triggered by CI/CD. It ingests build metrics via a webhook, compares the current build against historical baselines stored in **Airtable**, runs an **AI agent (GPT-4.1-mini)** to diagnose regressions and suggest optimizations, then posts results to **GitHub PR comments**, optionally emails alerts via **Gmail**, and archives the results back to Airtable.

**Primary use cases**
- Detect build-time regressions per PR/build.
- Identify “hotspot” modules/tasks that are getting slower.
- Create an audit trail of build performance and recommendations.
- Notify developers on PRs and escalate critical regressions via email.

### 1.1 Input Reception & Static Configuration
- Receives build metrics (buildId, repository, prNumber, totalDuration, gradle tasks, cocoapods pods, etc.).
- Initializes thresholds and exclusions used throughout analysis.

### 1.2 Historical Baseline Retrieval & Aggregation (Airtable)
- Fetches up to the last N builds (default 10) for the same PR + repository.
- Aggregates build-time statistics (avg/max/min) for baseline context.

### 1.3 Dataset Comparison & AI Input Preparation
- Builds a unified object containing current build + config + historical stats.
- Compares datasets and produces a “comparison” payload for the AI.

### 1.4 AI Analysis (Agent + Tools + Memory + Structured Output)
- AI agent uses GPT-4.1-mini and optional GitHub API tool to enrich context.
- Returns a structured JSON report (severity, regressions, root causes, recommendations, productivity impact).

### 1.5 Severity Routing & Notifications / Archival
- Routes by severity (Critical/Warning/Info).
- Posts PR comment, sends email on critical, archives record in Airtable.

---

## 2. Block-by-Block Analysis

### Block 1 — Webhook Trigger & Configuration

**Overview:** Receives CI build metrics via HTTP POST and sets analysis parameters (thresholds, exclusions, history depth).  
**Nodes involved:** `Webhook`, `Set Configuration`  

#### Node: Webhook
- **Type/role:** `Webhook` (n8n-nodes-base.webhook) — workflow entry point.
- **Configuration (interpreted):**
  - Method: **POST**
  - Path: **/build-hotspot-tracker**
  - Expects a JSON payload under `body` (as used by downstream expressions).
- **Key variables/fields consumed downstream:**
  - `$('Webhook').item.json.body.buildId`
  - `...body.prNumber`
  - `...body.repository` (expects `owner/repo` format, e.g., `repo/test_app`)
  - `...body.totalDuration`
  - `...body.gradle.tasks` (array with `module`, `duration`, `notCacheable`)
  - `...body.cocoapods.pods` (present but not used in current logic)
- **Outputs:** Main output to `Set Configuration`.
- **Potential failures / edge cases:**
  - Missing `body` or unexpected schema (expressions will fail).
  - `gradle.tasks` missing or not an array → expression in “Prepare AI Input” can error (`.length`).
  - Security: webhook endpoint should be protected (secret header, IP allowlist, or n8n auth) to prevent spoofed metrics.

#### Node: Set Configuration
- **Type/role:** `Set` (n8n-nodes-base.set) — defines constants used by the workflow.
- **Configuration (interpreted):**
  - Sets strings:
    - `regressionThreshold` = `"20"` (intended as percent threshold)
    - `historicalBuildsCount` = `"10"` (note: Airtable node hardcodes limit=10 separately)
    - `excludeModules` = `"test"` (intended to exclude test modules/tasks; not actually applied in filtering in this workflow)
- **Connections:**
  - Outputs to `Fetch Historical Builds`
  - Outputs to `Prepare AI Input`
- **Potential failures / edge cases:**
  - Values stored as strings; if numeric comparisons are later introduced, conversion may be required.
  - `excludeModules` is not referenced elsewhere; if you expect filtering, additional logic is needed.

---

### Block 2 — Historical Data Retrieval & Aggregation (Airtable)

**Overview:** Pulls recent historical builds for the same repository+PR and computes baseline stats (avg/max/min totalBuildTime).  
**Nodes involved:** `Fetch Historical Builds`, `Aggregate Historical Data`  

#### Node: Fetch Historical Builds
- **Type/role:** `Airtable` (n8n-nodes-base.airtable) — searches historical records.
- **Configuration (interpreted):**
  - Operation: **Search**
  - Base: `host Base` (ID: `appPH1qPPYCXj2YBO`)
  - Table: `tblBuildHistory` (ID: `tbl7PAEPfvn2C590h`)
  - Limit: **10**
  - Filter formula:
    - Matches records where `{repository}` equals webhook repository AND `{prNumber}` equals webhook prNumber
    - Formula used:
      - `AND({repository} = '{{ ... }}', {prNumber} = '{{ ... }}')`
- **Inputs:** From `Set Configuration`.
- **Outputs:** To `Aggregate Historical Data`.
- **Potential failures / edge cases:**
  - Airtable auth/token invalid, base/table permissions, rate limits.
  - If `prNumber` stored as number in Airtable but compared as string in formula (or vice versa), search may return nothing.
  - No historical rows → aggregator may output empty or null stats depending on node behavior/version.

#### Node: Aggregate Historical Data
- **Type/role:** `Aggregate` (n8n-nodes-base.aggregate) — aggregates fields across items.
- **Configuration (interpreted):**
  - Aggregates `totalBuildTime` into:
    - `avgBuildTime`
    - `maxBuildTime`
    - `minBuildTime`
- **Inputs:** Results from `Fetch Historical Builds`.
- **Outputs:** Feeds `Compare with Historical Builds` on input index **1**.
- **Potential failures / edge cases:**
  - If `totalBuildTime` missing/non-numeric in Airtable, aggregations may be incorrect or fail.
  - Empty input set may yield no output item; downstream compare node may not receive expected historicalStats.

---

### Block 3 — AI Input Packaging & Dataset Compare

**Overview:** Shapes current build + config into a standard payload and merges it with aggregated historical stats to produce a comparison dataset for the AI agent.  
**Nodes involved:** `Prepare AI Input`, `Compare with Historical Builds`  

#### Node: Prepare AI Input
- **Type/role:** `Set` (n8n-nodes-base.set) — prepares structured objects for later comparison/analysis.
- **Configuration (interpreted):** Creates 3 fields:
  - `currentBuild` = full webhook body (`$('Webhook').item.json.body`)
  - `config` = configuration object from Set Configuration (`$('Set Configuration').item.json`)
  - `buildMetrics` = object:
    - `buildId`, `prNumber`, `repository`, `totalDuration`
    - `gradleTasks` = `$('Webhook').item.json.body.gradle.tasks.length`
- **Inputs:** From `Set Configuration`.
- **Outputs:** To `Compare with Historical Builds` (input index **0**).
- **Potential failures / edge cases:**
  - If `body.gradle.tasks` is undefined, `.length` throws.
  - If the webhook sometimes reports only CocoaPods (no gradle), this node needs conditional guards.

#### Node: Compare with Historical Builds
- **Type/role:** `Compare Datasets` (n8n-nodes-base.compareDatasets) — merges/compares two inputs.
- **Configuration (interpreted):**
  - Merge by fields:
    - `buildId` (field1=buildId, field2=buildId)
  - Skip fields during compare:
    - `id, createdTime, timestamp`
  - Input 0: current build payload (from `Prepare AI Input`)
  - Input 1: aggregated stats (from `Aggregate Historical Data`)
- **Outputs:** To `AI Build Analyzer`.
- **Important behavior note:** The agent prompt references:
  - `...item.json.currentBuild`
  - `...item.json.historicalStats`
  - `...item.json.comparison`
  Those must exist in the Compare node output shape. If the Compare node is configured differently across versions or produces different property names, the AI prompt will break.
- **Potential failures / edge cases:**
  - No historical input item → compare output may omit `historicalStats`/`comparison`.
  - Mismatched join keys (`buildId`) because aggregated stats may not include buildId at all (aggregations typically collapse items). If so, “merge by buildId” may not behave as intended.
  - Version-specific: node v2.3 behaviors/field names can differ; validate output with a test execution.

---

### Block 4 — AI Analysis (Model, Agent, Output Parser, Memory, GitHub Tool)

**Overview:** Uses an AI agent powered by GPT-4.1-mini, with optional GitHub PR context via an HTTP tool, and returns a structured report used for routing and notifications.  
**Nodes involved:** `OpenAI Chat Model`, `AI Build Analyzer`, `Structured Output Parser`, `Build Context Memory`, `GitHub API Tool`  

#### Node: OpenAI Chat Model
- **Type/role:** `OpenAI Chat Model` (@n8n/n8n-nodes-langchain.lmChatOpenAi) — LLM provider for the agent.
- **Configuration (interpreted):**
  - Model: **gpt-4.1-mini**
  - Temperature: **0.2**
  - Max tokens: **2000**
- **Connection:** Supplies the language model to `AI Build Analyzer` via `ai_languageModel`.
- **Potential failures / edge cases:**
  - Invalid OpenAI credentials, insufficient quota, model not available in the account/region.
  - Token limit: large payloads from webhook/history can exceed context window; consider trimming.

#### Node: Structured Output Parser
- **Type/role:** Structured output parser (@n8n/n8n-nodes-langchain.outputParserStructured) — enforces JSON schema.
- **Configuration (interpreted):**
  - Manual JSON schema requiring:
    - `severity` ∈ {Critical, Warning, Info}
    - `regressions`: array of `{module, percentSlower, impact}`
    - `recommendations`: string array
    - Optional: `rootCauses`, `productivityImpact`
- **Connection:** Supplies output parser to `AI Build Analyzer` via `ai_outputParser`.
- **Potential failures / edge cases:**
  - If the model returns non-conforming JSON, parser will error and the workflow stops unless error handling is added.
  - Schema marks `rootCauses` and `productivityImpact` as optional, but downstream GitHub comment expects them; missing fields can break comment rendering unless guarded.

#### Node: Build Context Memory
- **Type/role:** Memory buffer window (@n8n/n8n-nodes-langchain.memoryBufferWindow) — conversational/session memory.
- **Configuration (interpreted):**
  - Session key: `{{ repository }}-{{ prNumber }}` derived from webhook body
  - Session ID type: custom key
- **Connection:** Supplies memory to `AI Build Analyzer` via `ai_memory`.
- **Potential failures / edge cases:**
  - If repository/prNumber missing, sessionKey becomes `undefined-undefined`.
  - Memory may persist across runs and could leak context between analyses if key collisions occur.

#### Node: GitHub API Tool
- **Type/role:** HTTP Request Tool (n8n-nodes-base.httpRequestTool) — callable tool for the agent.
- **Configuration (interpreted):**
  - URL template:
    - `https://api.github.com/repos/{repository}/pulls/{prNumber}`
    - Uses `$fromAI('repository', ...)` and `$fromAI('prNumber', ...)` so the agent decides tool inputs.
  - Adds header `Accept: application/vnd.github+json`
  - Auth: predefined credential type (GitHub credentials configured in node)
- **Connection:** Exposed to the agent via `ai_tool`.
- **Potential failures / edge cases:**
  - GitHub auth missing/invalid; rate limiting.
  - Agent might pass invalid repository format; URL becomes invalid and tool call fails.
  - Tool is described as also fetching “file changes”, but the endpoint configured returns PR details; file changes would require `/pulls/{prNumber}/files`.

#### Node: AI Build Analyzer
- **Type/role:** LangChain Agent (@n8n/n8n-nodes-langchain.agent) — orchestrates analysis.
- **Configuration (interpreted):**
  - System message: positions the agent as a Gradle/CocoaPods build performance expert; must classify severity and estimate productivity impact.
  - User prompt includes three injected fields from Compare node output:
    - Current Build
    - Historical Statistics
    - Comparison
  - Includes instruction: “regressions (modules >20% slower)” (note: threshold hardcoded in prompt; it does not use `regressionThreshold` from config).
  - Output parser enabled (must match schema).
- **Inputs:** From `Compare with Historical Builds`.
- **Outputs:** Main output to `Switch`.
- **Potential failures / edge cases:**
  - If Compare node doesn’t provide the referenced fields, prompt will embed `undefined`.
  - Hardcoded “>20%” may diverge from `Set Configuration.regressionThreshold`.
  - Parser errors stop execution; consider fallback behavior.

---

### Block 5 — Severity Routing, Reporting, Alerts, and Archiving

**Overview:** Routes based on AI severity and performs actions: comment on PR, send email for critical, and store a record in Airtable.  
**Nodes involved:** `Switch`, `Comment on PR`, `Notify Email`, `Store Build Data`  

#### Node: Switch
- **Type/role:** `Switch` (n8n-nodes-base.switch) — routes based on `$json.severity`.
- **Configuration (interpreted):**
  - 3 rules (renamed outputs):
    - “Critical” if `$json.severity == "Critical"`
    - “Warning” if `$json.severity == "Warning"`
    - “Info” if `$json.severity == "Info"`
- **Connections (as configured):**
  - Output 0 (Critical): `Store Build Data`, `Comment on PR`, `Notify Email`
  - Output 1 (Warning): `Comment on PR`
  - Output 2 (Info): `Store Build Data`
- **Potential failures / edge cases:**
  - If severity missing or not one of the three values, nothing runs (no default path configured).
  - Case sensitivity is enabled; “critical” would not match.

#### Node: Comment on PR
- **Type/role:** `GitHub` (n8n-nodes-base.github) — posts a PR/issue comment.
- **Configuration (interpreted):**
  - Operation: Create comment
  - Owner: `repo`
  - Repository: `test_app`
  - Issue number: `$('AI Build Analyzer').item.json.prNumber`
  - Comment body is a formatted report using:
    - `severity`, `productivityImpact`
    - `regressions` mapped into bullet list
    - `rootCauses` and `recommendations`
- **Inputs:** From `Switch` (Critical and Warning paths).
- **Potential failures / edge cases:**
  - Owner/repo are hardcoded and may not match webhook `repository`. If webhook repo differs, comments will go to the wrong repo or fail.
  - If `rootCauses` or `productivityImpact` absent, expression can fail (e.g., calling `.map` on undefined).
  - GitHub permissions: requires write access to issues/PR comments.

#### Node: Notify Email
- **Type/role:** `Gmail` (n8n-nodes-base.gmail) — sends alert emails on critical regressions.
- **Configuration (interpreted):**
  - To: `user@example.com` (placeholder)
  - Subject: `🚨 Build Regression Detected - PR #{{ $('AI Build Analyzer').item.json.regressions[0].module }}`
    - Note: subject references `regressions[0].module` but labels it as PR #; likely a bug.
  - Message body includes:
    - PR number and repository from `Webhook`
    - regressions list and recommendations from `AI Build Analyzer`
    - PR link: `https://github.com/{{ repository }}/pull/{{ prNumber }}`
- **Inputs:** From `Switch` (Critical path only).
- **Potential failures / edge cases:**
  - If `regressions` is empty, `regressions[0]` is undefined → expression failure in subject.
  - Repository formatting: webhook uses `repo/test_app`, link expects `owner/repo`; OK if consistent.
  - Gmail OAuth scope/refresh token failures.

#### Node: Store Build Data
- **Type/role:** `Airtable` (n8n-nodes-base.airtable) — archives build analysis results.
- **Configuration (interpreted):**
  - Operation: **Create record**
  - Base: `host Base` (appPH1qPPYCXj2YBO)
  - Table: `tblBuildHistory` (tbl7PAEPfvn2C590h)
  - Fields written:
    - buildId, prNumber, repository (from webhook)
    - timestamp = `$now.toISO()`
    - totalBuildTime = webhook `totalDuration`
    - slowestModules = join of AI regressions as `"module: X% slower"`
    - recommendations = AI recommendations joined with `; `
- **Inputs:** From `Switch` (Critical and Info paths).
- **Potential failures / edge cases:**
  - If regressions undefined/empty, `.map(...)` may fail unless regressions always present (schema requires it, but it can still be empty array).
  - Airtable type mismatches (e.g., totalBuildTime expects number).
  - Duplicate records: no upsert/dedup logic; repeated webhook calls create duplicates.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook | Webhook | Entry point; receives build metrics from CI | — | Set Configuration | ## Webhook Trigger on Push/PR & Set Configuration  \nThe Webhook node is the workflow's entry point, listening for a POST request at /webhook/build-hotspot-tracker sent by the CI/CD job upon build completion. It receives the critical JSON payload, including build ID, PR number, and task durations.  \n\nThe subsequent Set Configuration node initializes static parameters for the analysis. This includes the regressionThreshold (default 20 or 20%) used to identify slow modules and excludeModules (default test), which filters out irrelevant tasks like unit tests from the performance calculation. |
| Set Configuration | Set | Defines thresholds/exclusions/history count | Webhook | Fetch Historical Builds; Prepare AI Input | ## Webhook Trigger on Push/PR & Set Configuration  \nThe Webhook node is the workflow's entry point, listening for a POST request at /webhook/build-hotspot-tracker sent by the CI/CD job upon build completion. It receives the critical JSON payload, including build ID, PR number, and task durations.  \n\nThe subsequent Set Configuration node initializes static parameters for the analysis. This includes the regressionThreshold (default 20 or 20%) used to identify slow modules and excludeModules (default test), which filters out irrelevant tasks like unit tests from the performance calculation. |
| Fetch Historical Builds | Airtable | Query historical build records for same repo+PR | Set Configuration | Aggregate Historical Data | ## Historical Data Aggregation & AI Preparation  \nFetches the last 10 builds from Airtable to calculate average, maximum, and minimum performance baselines.  \nStructures current build metrics and configuration parameters into a standardized format for processing.  \nMerges real-time data with historical statistics to create a unified dataset for comparative analysis. |
| Aggregate Historical Data | Aggregate | Compute avg/max/min baseline build times | Fetch Historical Builds | Compare with Historical Builds | ## Historical Data Aggregation & AI Preparation  \nFetches the last 10 builds from Airtable to calculate average, maximum, and minimum performance baselines.  \nStructures current build metrics and configuration parameters into a standardized format for processing.  \nMerges real-time data with historical statistics to create a unified dataset for comparative analysis. |
| Prepare AI Input | Set | Standardize current build + config for AI | Set Configuration | Compare with Historical Builds | ## Historical Data Aggregation & AI Preparation  \nFetches the last 10 builds from Airtable to calculate average, maximum, and minimum performance baselines.  \nStructures current build metrics and configuration parameters into a standardized format for processing.  \nMerges real-time data with historical statistics to create a unified dataset for comparative analysis. |
| Compare with Historical Builds | Compare Datasets | Merge current build with historical stats into comparison payload | Prepare AI Input; Aggregate Historical Data | AI Build Analyzer | ## Historical Data Aggregation & AI Preparation  \nFetches the last 10 builds from Airtable to calculate average, maximum, and minimum performance baselines.  \nStructures current build metrics and configuration parameters into a standardized format for processing.  \nMerges real-time data with historical statistics to create a unified dataset for comparative analysis. |
| OpenAI Chat Model | OpenAI Chat Model (LangChain) | Provides GPT-4.1-mini to the agent | — | AI Build Analyzer (ai_languageModel) | ## AI-Powered Analysis & Severity Routing  \nIntelligent Diagnosis: Uses an AI Agent (GPT-4o-mini) to identify build regressions (>20% slowdown), determine root causes, and suggest optimizations.  \nContextual Awareness: Integrates the GitHub API and session memory to factor in PR details and historical build context.  \nStructured Routing: Converts AI insights into a standardized JSON format and routes the workflow based on severity (Critical, Warning, or Info). |
| Build Context Memory | Memory Buffer Window (LangChain) | Keeps session memory per repo+PR | — | AI Build Analyzer (ai_memory) | ## AI-Powered Analysis & Severity Routing  \nIntelligent Diagnosis: Uses an AI Agent (GPT-4o-mini) to identify build regressions (>20% slowdown), determine root causes, and suggest optimizations.  \nContextual Awareness: Integrates the GitHub API and session memory to factor in PR details and historical build context.  \nStructured Routing: Converts AI insights into a standardized JSON format and routes the workflow based on severity (Critical, Warning, or Info). |
| GitHub API Tool | HTTP Request Tool | Tool callable by agent to fetch PR context | — | AI Build Analyzer (ai_tool) | ## AI-Powered Analysis & Severity Routing  \nIntelligent Diagnosis: Uses an AI Agent (GPT-4o-mini) to identify build regressions (>20% slowdown), determine root causes, and suggest optimizations.  \nContextual Awareness: Integrates the GitHub API and session memory to factor in PR details and historical build context.  \nStructured Routing: Converts AI insights into a standardized JSON format and routes the workflow based on severity (Critical, Warning, or Info). |
| Structured Output Parser | Structured Output Parser (LangChain) | Enforces JSON schema for agent output | — | AI Build Analyzer (ai_outputParser) | ## AI-Powered Analysis & Severity Routing  \nIntelligent Diagnosis: Uses an AI Agent (GPT-4o-mini) to identify build regressions (>20% slowdown), determine root causes, and suggest optimizations.  \nContextual Awareness: Integrates the GitHub API and session memory to factor in PR details and historical build context.  \nStructured Routing: Converts AI insights into a standardized JSON format and routes the workflow based on severity (Critical, Warning, or Info). |
| AI Build Analyzer | LangChain Agent | Produces severity/regressions/root causes/recommendations | Compare with Historical Builds; (uses OpenAI Chat Model, Memory, Tool, Parser) | Switch | ## AI-Powered Analysis & Severity Routing  \nIntelligent Diagnosis: Uses an AI Agent (GPT-4o-mini) to identify build regressions (>20% slowdown), determine root causes, and suggest optimizations.  \nContextual Awareness: Integrates the GitHub API and session memory to factor in PR details and historical build context.  \nStructured Routing: Converts AI insights into a standardized JSON format and routes the workflow based on severity (Critical, Warning, or Info). |
| Switch | Switch | Route actions by severity | AI Build Analyzer | (Critical) Store Build Data, Comment on PR, Notify Email; (Warning) Comment on PR; (Info) Store Build Data | ## AI-Powered Analysis & Severity Routing  \nIntelligent Diagnosis: Uses an AI Agent (GPT-4o-mini) to identify build regressions (>20% slowdown), determine root causes, and suggest optimizations.  \nContextual Awareness: Integrates the GitHub API and session memory to factor in PR details and historical build context.  \nStructured Routing: Converts AI insights into a standardized JSON format and routes the workflow based on severity (Critical, Warning, or Info). |
| Store Build Data | Airtable | Archive analysis results and build metrics | Switch | — | ## Automated Reporting & Data Archiving  \nGitHub Feedback: Posts a detailed performance report directly to the Pull Request, outlining regressions, root causes, and optimization steps.  \nCritical Alerts: Sends real-time Gmail notifications to the team when significant build slowdowns are detected, including direct links to the PR.  \nHistory Log: Archives all build metrics, detected "hotspots," and AI suggestions in Airtable for long-term trend analysis and auditing. |
| Comment on PR | GitHub | Post analysis report as PR comment | Switch | — | ## Automated Reporting & Data Archiving  \nGitHub Feedback: Posts a detailed performance report directly to the Pull Request, outlining regressions, root causes, and optimization steps.  \nCritical Alerts: Sends real-time Gmail notifications to the team when significant build slowdowns are detected, including direct links to the PR.  \nHistory Log: Archives all build metrics, detected "hotspots," and AI suggestions in Airtable for long-term trend analysis and auditing. |
| Notify Email | Gmail | Email alert for critical regressions | Switch | — | ## Automated Reporting & Data Archiving  \nGitHub Feedback: Posts a detailed performance report directly to the Pull Request, outlining regressions, root causes, and optimization steps.  \nCritical Alerts: Sends real-time Gmail notifications to the team when significant build slowdowns are detected, including direct links to the PR.  \nHistory Log: Archives all build metrics, detected "hotspots," and AI suggestions in Airtable for long-term trend analysis and auditing. |
| Sticky Note10 | Sticky Note | Global explanation & setup notes | — | — | # How It Works  \nThis workflow serves as an automated CI/CD Build Performance Monitor, analyzing metrics from Gradle or CocoaPods builds to detect regressions and identify performance "hotspots". It is initiated by a Webhook that receives a JSON payload containing task durations, module names, and build context. The system uses historical data from Airtable and an AI Agent (GPT-4o-mini) to compare current performance against baselines, categorize the severity of any slowdowns, and generate actionable optimization recommendations.  \n# Setup Steps  \n## Webhook Trigger  \nConfigure your CI/CD pipeline (e.g., GitHub Actions, Jenkins) to send a POST request with build metrics to the workflow's Webhook URL.  \n\n## Connect Accounts  \nAirtable: Add a Personal Access Token to allow the workflow to fetch historical data and store new analysis records.  \nGitHub: Provide API credentials with repository write access to enable automated commenting on Pull Requests.  \nGmail: Set up OAuth2 credentials to send automated email alerts when critical performance regressions are detected.  \nOpenAI: Ensure active credits are available for the AI Build Analyzer node to perform the regression analysis.  \n\n## Configure Parameters  \nSet Configuration: Adjust the regressionThreshold (default 20%) and define excludeModules (e.g., filtering out "test" tasks) to tailor analysis to your project.  \n\nStorage Nodes: Verify the Airtable Base and Table IDs in both the "Fetch Historical Builds" and "Store Build Data" nodes to ensure data is routed to your specific tracking table.  \n\n## Update Notifications  \nComment on PR: Confirm that the GitHub account used has the necessary permissions to post comments on the target repository.  \n\nNotify Email: Update the recipient email address in the Gmail node to ensure alerts reach the correct team members. |
| Sticky Note3 | Sticky Note | Documentation (section note) | — | — | ## Webhook Trigger on Push/PR & Set Configuration  \nThe Webhook node is the workflow's entry point, listening for a POST request at /webhook/build-hotspot-tracker sent by the CI/CD job upon build completion. It receives the critical JSON payload, including build ID, PR number, and task durations.  \n\nThe subsequent Set Configuration node initializes static parameters for the analysis. This includes the regressionThreshold (default 20 or 20%) used to identify slow modules and excludeModules (default test), which filters out irrelevant tasks like unit tests from the performance calculation. |
| Sticky Note | Sticky Note | Documentation (section note) | — | — | ## Historical Data Aggregation & AI Preparation  \nFetches the last 10 builds from Airtable to calculate average, maximum, and minimum performance baselines.  \nStructures current build metrics and configuration parameters into a standardized format for processing.  \nMerges real-time data with historical statistics to create a unified dataset for comparative analysis. |
| Sticky Note4 | Sticky Note | Documentation (section note) | — | — | ## AI-Powered Analysis & Severity Routing  \nIntelligent Diagnosis: Uses an AI Agent (GPT-4o-mini) to identify build regressions (>20% slowdown), determine root causes, and suggest optimizations.  \nContextual Awareness: Integrates the GitHub API and session memory to factor in PR details and historical build context.  \nStructured Routing: Converts AI insights into a standardized JSON format and routes the workflow based on severity (Critical, Warning, or Info). |
| Sticky Note5 | Sticky Note | Documentation (section note) | — | — | ## Automated Reporting & Data Archiving  \nGitHub Feedback: Posts a detailed performance report directly to the Pull Request, outlining regressions, root causes, and optimization steps.  \nCritical Alerts: Sends real-time Gmail notifications to the team when significant build slowdowns are detected, including direct links to the PR.  \nHistory Log: Archives all build metrics, detected "hotspots," and AI suggestions in Airtable for long-term trend analysis and auditing. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named:  
   `Build Time Hotspot Tracker - Gradle/CocoaPods Analyzer-f`

2. **Add Trigger Node: Webhook**
   - Node type: **Webhook**
   - HTTP Method: **POST**
   - Path: **build-hotspot-tracker**
   - Ensure it accepts JSON (default).
   - (Recommended) Add security: header token check or n8n webhook auth.

3. **Add Node: Set Configuration**
   - Node type: **Set**
   - Add fields (String):
     - `regressionThreshold` = `20`
     - `historicalBuildsCount` = `10`
     - `excludeModules` = `test`
   - Connect: `Webhook → Set Configuration`

4. **Add Node: Fetch Historical Builds (Airtable Search)**
   - Node type: **Airtable**
   - Operation: **Search**
   - Credentials: **Airtable Personal Access Token**
   - Base: select your base (e.g., `host Base`)
   - Table: select your table (e.g., `tblBuildHistory`)
   - Limit: `10`
   - Filter by formula:
     - `AND({repository} = '{{ $('Webhook').item.json.body.repository }}', {prNumber} = '{{ $('Webhook').item.json.body.prNumber }}')`
   - Connect: `Set Configuration → Fetch Historical Builds`

5. **Add Node: Aggregate Historical Data**
   - Node type: **Aggregate**
   - Aggregate field: `totalBuildTime`
   - Output fields:
     - `avgBuildTime` (average of totalBuildTime)
     - `maxBuildTime` (max of totalBuildTime)
     - `minBuildTime` (min of totalBuildTime)
   - Connect: `Fetch Historical Builds → Aggregate Historical Data`

6. **Add Node: Prepare AI Input**
   - Node type: **Set**
   - Add assignments:
     - `currentBuild` (Object) = `$('Webhook').item.json.body`
     - `config` (Object) = `$('Set Configuration').item.json`
     - `buildMetrics` (Object) =
       - `buildId`: `$('Webhook').item.json.body.buildId`
       - `prNumber`: `$('Webhook').item.json.body.prNumber`
       - `repository`: `$('Webhook').item.json.body.repository`
       - `totalDuration`: `$('Webhook').item.json.body.totalDuration`
       - `gradleTasks`: `$('Webhook').item.json.body.gradle.tasks.length`
   - Connect: `Set Configuration → Prepare AI Input`

7. **Add Node: Compare with Historical Builds**
   - Node type: **Compare Datasets**
   - Configure “merge by fields”:
     - field1 `buildId`, field2 `buildId`
   - Options: skip fields `id, createdTime, timestamp`
   - Connect inputs:
     - Input 0: `Prepare AI Input → Compare with Historical Builds`
     - Input 1: `Aggregate Historical Data → Compare with Historical Builds`

8. **Add AI Components**
   1) **OpenAI Chat Model**
   - Node type: **OpenAI Chat Model (LangChain)**
   - Credentials: **OpenAI API**
   - Model: `gpt-4.1-mini`
   - Temperature: `0.2`
   - Max tokens: `2000`

   2) **Structured Output Parser**
   - Node type: **Structured Output Parser (LangChain)**
   - Schema type: Manual
   - Paste the schema that enforces:
     - severity enum (Critical/Warning/Info)
     - regressions array of objects
     - recommendations array of strings
     - rootCauses/productivityImpact optional

   3) **Build Context Memory**
   - Node type: **Memory Buffer Window**
   - Session key: `{{ $('Webhook').item.json.body.repository }}-{{ $('Webhook').item.json.body.prNumber }}`
   - Session ID type: custom key

   4) **GitHub API Tool**
   - Node type: **HTTP Request Tool**
   - Authentication: GitHub predefined credential type (configure a GitHub token/OAuth credential)
   - URL:
     - `https://api.github.com/repos/{{ $fromAI('repository', 'GitHub repository in format owner/repo', 'string') }}/pulls/{{ $fromAI('prNumber', 'Pull request number', 'string') }}`
   - Headers: `Accept: application/vnd.github+json`

9. **Add Node: AI Build Analyzer (Agent)**
   - Node type: **AI Agent (LangChain)**
   - Prompt type: define
   - System message: build performance analyst (as in workflow)
   - User text references the Compare node output:
     - currentBuild, historicalStats, comparison
   - Enable output parser.
   - Connect:
     - `Compare with Historical Builds → AI Build Analyzer` (main)
     - `OpenAI Chat Model → AI Build Analyzer` (ai_languageModel)
     - `Structured Output Parser → AI Build Analyzer` (ai_outputParser)
     - `Build Context Memory → AI Build Analyzer` (ai_memory)
     - `GitHub API Tool → AI Build Analyzer` (ai_tool)

10. **Add Node: Switch (Severity Router)**
    - Node type: **Switch**
    - Rules:
      - If `{{ $json.severity }}` equals `Critical` → output “Critical”
      - equals `Warning` → output “Warning”
      - equals `Info` → output “Info”
    - Connect: `AI Build Analyzer → Switch`

11. **Add Node: Comment on PR (GitHub)**
    - Node type: **GitHub**
    - Credentials: GitHub with repo write access
    - Operation: **Create comment**
    - Owner: set to your owner (note: current workflow hardcodes `repo`)
    - Repository: set to your repo (note: current workflow hardcodes `test_app`)
    - Issue number: `{{ $('AI Build Analyzer').item.json.prNumber }}`
    - Body: compose with fields from AI output (severity, regressions, root causes, recommendations, productivityImpact)
    - Connect:
      - Switch “Critical” output → Comment on PR
      - Switch “Warning” output → Comment on PR

12. **Add Node: Notify Email (Gmail)**
    - Node type: **Gmail**
    - Credentials: **Gmail OAuth2**
    - Send To: your team address
    - Subject/body: reference webhook + AI output (ensure safe handling if regressions is empty)
    - Connect:
      - Switch “Critical” output → Notify Email

13. **Add Node: Store Build Data (Airtable Create)**
    - Node type: **Airtable**
    - Operation: **Create**
    - Same base/table as history
    - Map fields:
      - buildId, prNumber, repository, totalBuildTime, timestamp
      - slowestModules from AI regressions
      - recommendations joined
    - Connect:
      - Switch “Critical” output → Store Build Data
      - Switch “Info” output → Store Build Data

14. **Test end-to-end**
    - Send a sample POST to webhook with:
      - `buildId`, `prNumber`, `repository`, `totalDuration`
      - `gradle.tasks` array (even if empty)
    - Verify:
      - Airtable search returns items
      - Compare node output shape matches AI prompt references
      - AI output validates against schema
      - Switch routes correctly

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Disclaimer (provided): Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques. | Context: Compliance / provenance statement |
| Webhook must be called by CI/CD with JSON payload including buildId, prNumber, repository, task durations. | Context: Sticky Note “How It Works / Setup Steps” |
| Ensure credentials: Airtable PAT, GitHub write access, Gmail OAuth2, OpenAI credits. | Context: Sticky Note “How It Works / Setup Steps” |
| Validate Airtable Base/Table IDs in both history fetch and storage nodes. | Context: Sticky Note “How It Works / Setup Steps” |
| Update email recipient in Gmail node (`user@example.com`). | Context: Sticky Note “How It Works / Setup Steps” |


Detect misinformation and manipulation risks with GPT-4o agents and Google Sheets

https://n8nworkflows.xyz/workflows/detect-misinformation-and-manipulation-risks-with-gpt-4o-agents-and-google-sheets-14002


# Detect misinformation and manipulation risks with GPT-4o agents and Google Sheets

# 1. Workflow Overview

This workflow analyzes social-media post datasets for misinformation, coordinated manipulation, and bot-like amplification patterns using a multi-agent AI design built on GPT-4o and Python-based code tools. It ends by formatting the structured risk assessment and appending it to Google Sheets for tracking and audit.

Typical use cases include platform trust and safety operations, media intelligence, moderation support, and research teams investigating disinformation campaigns.

## 1.1 Entry and Orchestration

The workflow starts manually and sends the incoming dataset to a supervisor agent. This supervisor does not perform all analysis directly; instead, it delegates specialized tasks to three agent-tools and combines their outputs into one structured assessment.

## 1.2 Narrative Analysis

A dedicated narrative-analysis agent detects coordinated narratives and thematic clustering across posts. It uses a Python semantic clustering tool based on TF-IDF and cosine similarity.

## 1.3 Bot and Propagation Analysis

A second agent investigates bot-like behavior and coordinated amplification. It relies on three Python tools: one for repost/share cascade modeling, one for temporal posting pattern analysis, and one for generating a risk heatmap from combined narrative and bot findings.

## 1.4 Manipulation Technique Classification

A third agent classifies manipulation tactics present in post content. It uses a taxonomy-driven Python tool to identify emotional appeals, logical fallacies, propaganda techniques, and disinformation patterns.

## 1.5 Structured Output, Formatting, and Storage

The supervisor output is forced into a structured JSON shape by an output parser. The workflow then flattens key fields, serializes the full analysis, and appends the result to Google Sheets.

---

# 2. Block-by-Block Analysis

## 2.1 Block: Manual Trigger and Supervisor Orchestration

### Overview
This block starts the workflow and hands the post dataset to the central AI supervisor. The supervisor coordinates the specialist agents, requests their analyses as tools, and produces the final risk assessment in a predefined schema.

### Nodes Involved
- Start Analysis
- Misinformation Detection Supervisor
- Supervisor Model
- Risk Assessment Output Parser

### Node Details

#### Start Analysis
- **Type and role:** `n8n-nodes-base.manualTrigger`; manual execution entry point.
- **Configuration choices:** No parameters are configured. It requires a human to click execute in n8n.
- **Key expressions or variables used:** None.
- **Input and output connections:** No input; outputs to **Misinformation Detection Supervisor**.
- **Version-specific requirements:** Type version 1.
- **Edge cases / failures:** None at runtime beyond ordinary manual execution behavior. The main practical limitation is that it provides no native input payload unless execution data is manually supplied or upstream test data is injected.
- **Sub-workflow reference:** None.

#### Misinformation Detection Supervisor
- **Type and role:** `@n8n/n8n-nodes-langchain.agent`; central orchestration agent.
- **Configuration choices:**
  - Prompt mode is explicitly defined.
  - Input text is `{{$json.posts}}`, so it expects the incoming item to contain a `posts` field.
  - `maxIterations` is set to 5, limiting tool-calling loops.
  - System message instructs the agent to coordinate:
    - Narrative Pattern Detector Agent
    - Bot Behavior Analyzer Agent
    - Manipulation Technique Classifier Agent
  - It is configured with an output parser to force structured output.
- **Key expressions or variables used:**
  - `={{ $json.posts }}`
- **Input and output connections:**
  - Main input from **Start Analysis**
  - AI language model from **Supervisor Model**
  - AI tools from:
    - **Narrative Pattern Detector Agent**
    - **Bot Behavior Analyzer Agent**
    - **Manipulation Technique Classifier Agent**
  - AI output parser from **Risk Assessment Output Parser**
  - Main output to **Format Results**
- **Version-specific requirements:** Type version 3.1; requires n8n LangChain-compatible agent support.
- **Edge cases / failures:**
  - Fails if `posts` is missing from the incoming JSON.
  - Can produce incomplete output if agent iterations are insufficient.
  - Tool invocation may fail if sub-agents return malformed responses.
  - OpenAI credential or rate-limit issues can interrupt execution.
  - Structured parser may reject outputs that do not conform to the schema.
- **Sub-workflow reference:** None; this is not an Execute Workflow node, but it orchestrates tool-agents internally.

#### Supervisor Model
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; GPT-4o chat model for the supervisor agent.
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0.3`
  - No built-in tools enabled in the model node itself.
- **Key expressions or variables used:** None.
- **Input and output connections:** Connected to **Misinformation Detection Supervisor** via `ai_languageModel`.
- **Version-specific requirements:** Type version 1.3.
- **Edge cases / failures:**
  - Missing or invalid OpenAI credentials.
  - Model access restrictions on the account.
  - Token/context overrun if the incoming dataset is too large.
  - API latency or rate limiting.
- **Sub-workflow reference:** None.

#### Risk Assessment Output Parser
- **Type and role:** `@n8n/n8n-nodes-langchain.outputParserStructured`; validates and shapes final supervisor output.
- **Configuration choices:**
  - Uses a JSON schema example that enforces a structure including:
    - `overall_risk_level`
    - `narrative_analysis`
    - `bot_analysis`
    - `discourse_risk_heatmap`
    - `recommendations`
- **Key expressions or variables used:** None.
- **Input and output connections:** Connected to **Misinformation Detection Supervisor** as `ai_outputParser`.
- **Version-specific requirements:** Type version 1.3.
- **Edge cases / failures:**
  - Schema mismatch if the supervisor omits required nested fields.
  - Enum mismatch if `overall_risk_level` is not one of `critical`, `high`, `moderate`, `low`.
  - Arrays or nested objects may be missing if sub-agent responses are weakly grounded.
- **Sub-workflow reference:** None.

---

## 2.2 Block: Narrative Pattern Detection

### Overview
This block identifies recurring narratives across social posts. It combines a GPT-4o sub-agent with a Python tool that clusters posts semantically using TF-IDF and cosine similarity.

### Nodes Involved
- Narrative Pattern Detector Agent
- Narrative Detector Model
- Semantic Clustering Tool

### Node Details

#### Narrative Pattern Detector Agent
- **Type and role:** `@n8n/n8n-nodes-langchain.agentTool`; sub-agent exposed to the supervisor as a callable tool.
- **Configuration choices:**
  - Its input text is sourced from the parent AI context using:
    `{{$fromAI('post_dataset', 'Social media post dataset to analyze for narrative patterns')}}`
  - System message instructs it to identify:
    - coordinated narratives
    - manipulation techniques
    - narrative coherence and coordination indicators
  - Tool description explains its purpose to the supervisor.
- **Key expressions or variables used:**
  - `$fromAI('post_dataset', ...)`
- **Input and output connections:**
  - AI language model from **Narrative Detector Model**
  - AI tool from **Semantic Clustering Tool**
  - Exposed upward to **Misinformation Detection Supervisor** as an `ai_tool`
- **Version-specific requirements:** Type version 3.
- **Edge cases / failures:**
  - The supervisor may pass data in a shape the tool cannot interpret.
  - If post objects lack a `content` field, the clustering tool will degrade or return weak output.
  - The agent prompt mentions manipulation techniques, but its attached tool primarily clusters semantics rather than classifying manipulative rhetoric directly.
- **Sub-workflow reference:** None.

#### Narrative Detector Model
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; GPT-4o model for the narrative sub-agent.
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0.2`
- **Key expressions or variables used:** None.
- **Input and output connections:** Connected to **Narrative Pattern Detector Agent** via `ai_languageModel`.
- **Version-specific requirements:** Type version 1.3.
- **Edge cases / failures:**
  - OpenAI auth and rate-limit errors.
  - Context-size issues if the dataset is large.
- **Sub-workflow reference:** None.

#### Semantic Clustering Tool
- **Type and role:** `@n8n/n8n-nodes-langchain.toolCode`; Python analysis tool for semantic clustering.
- **Configuration choices:**
  - Language: Python
  - Expects AI-provided `posts` data via:
    `$fromAI("posts", "Array of social media posts with content field", "object")`
  - Implements:
    - tokenization with regex
    - TF-IDF computation
    - cosine similarity
    - simple threshold-based clustering (`threshold=0.3`)
    - cluster theme extraction from top TF-IDF terms
  - Reports only clusters with at least 2 posts.
- **Key expressions or variables used:**
  - `$fromAI("posts", ..., "object")`
- **Input and output connections:** Connected to **Narrative Pattern Detector Agent** as `ai_tool`.
- **Version-specific requirements:** Type version 1.3; requires n8n code tool support for Python execution.
- **Edge cases / failures:**
  - Empty `content` values can generate zero vectors and low-quality clusters.
  - If all posts have highly unique vocabulary, clustering will be sparse.
  - JSON parsing fails if the AI sends malformed JSON strings.
  - TF-IDF implementation increments document frequency per token occurrence rather than unique-term-per-document logic, which may skew scoring.
  - Theme extraction may produce weak labels from noisy tokens.
- **Sub-workflow reference:** None.

---

## 2.3 Block: Bot Behavior and Amplification Analysis

### Overview
This block detects suspicious amplification behavior and generates explainable risk indicators. It uses a GPT-4o sub-agent plus three Python tools covering propagation cascades, temporal anomalies, and heatmap synthesis.

### Nodes Involved
- Bot Behavior Analyzer Agent
- Bot Analyzer Model
- Propagation Cascade Analyzer
- Temporal Pattern Detector
- Risk Heatmap Generator

### Node Details

#### Bot Behavior Analyzer Agent
- **Type and role:** `@n8n/n8n-nodes-langchain.agentTool`; supervisor-callable sub-agent for bot and spread analysis.
- **Configuration choices:**
  - Input is pulled from parent AI context using:
    `{{$fromAI('activity_dataset', 'User activity and propagation data to analyze for bot behavior')}}`
  - System message instructs it to detect:
    - bot-like behavioral signatures
    - propagation cascades
    - temporal activity anomalies
    - discourse risk heatmap with explainability
- **Key expressions or variables used:**
  - `$fromAI('activity_dataset', ...)`
- **Input and output connections:**
  - AI language model from **Bot Analyzer Model**
  - AI tools from:
    - **Propagation Cascade Analyzer**
    - **Temporal Pattern Detector**
    - **Risk Heatmap Generator**
  - Exposed upward to **Misinformation Detection Supervisor** as `ai_tool`
- **Version-specific requirements:** Type version 3.
- **Edge cases / failures:**
  - Requires post/activity records with IDs, timestamps, and ideally repost-parent relationships.
  - The heatmap tool expects both bot and narrative results, so the agent must orchestrate tool outputs carefully.
  - Missing timestamps or malformed ISO dates reduce accuracy.
- **Sub-workflow reference:** None.

#### Bot Analyzer Model
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; GPT-4o model for bot analysis.
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0.2`
- **Key expressions or variables used:** None.
- **Input and output connections:** Connected to **Bot Behavior Analyzer Agent** via `ai_languageModel`.
- **Version-specific requirements:** Type version 1.3.
- **Edge cases / failures:**
  - OpenAI auth, rate-limit, latency, or context-window issues.
- **Sub-workflow reference:** None.

#### Propagation Cascade Analyzer
- **Type and role:** `@n8n/n8n-nodes-langchain.toolCode`; Python tool for repost/share cascade modeling.
- **Configuration choices:**
  - Expects `activity_data` from AI context.
  - Builds a propagation graph from `id`/`post_id`, `repost_of`/`parent_id`, and timestamp fields.
  - Uses BFS to compute:
    - cascade depth
    - total posts
    - spread velocity
  - Reports only cascades with at least 3 posts.
  - Returns up to 10 cascades sorted by velocity.
- **Key expressions or variables used:**
  - `$fromAI("activity_data", "Array of posts with repost/share relationships and timestamps", "object")`
- **Input and output connections:** Connected to **Bot Behavior Analyzer Agent** as `ai_tool`.
- **Version-specific requirements:** Type version 1.3.
- **Edge cases / failures:**
  - Posts without IDs cannot be modeled correctly.
  - Bad timestamps fall back to current time, potentially corrupting velocity calculations.
  - Multiple disconnected graphs are supported, but only origins with children are analyzed.
  - Very small datasets may produce no cascades.
- **Sub-workflow reference:** None.

#### Temporal Pattern Detector
- **Type and role:** `@n8n/n8n-nodes-langchain.toolCode`; Python tool for user-level temporal anomaly scoring.
- **Configuration choices:**
  - Expects `user_activity` from AI context.
  - Groups posts by `user_id` or `author_id`.
  - Computes:
    - average posting interval
    - coefficient of variation
    - burst ratio
    - regularity score
    - hour diversity
    - posts per hour
  - Produces `bot_probability`, behavioral patterns, anomalies, and metrics.
  - Only suspicious accounts with `bot_probability > 0.3` are returned.
- **Key expressions or variables used:**
  - `$fromAI("user_activity", "Array of user posts with timestamps and user IDs", "object")`
- **Input and output connections:** Connected to **Bot Behavior Analyzer Agent** as `ai_tool`.
- **Version-specific requirements:** Type version 1.3.
- **Edge cases / failures:**
  - Users with fewer than 3 posts are skipped.
  - If time span is near zero, `posts_per_hour` can become extremely high or unstable.
  - Invalid timestamps default to now, reducing signal quality.
  - Heuristic thresholds are simplistic and may generate false positives for newsrooms, live-event accounts, or automation tools.
- **Sub-workflow reference:** None.

#### Risk Heatmap Generator
- **Type and role:** `@n8n/n8n-nodes-langchain.toolCode`; Python tool that synthesizes explainable risk indicators from narrative and bot outputs.
- **Configuration choices:**
  - Expects:
    - `narrative_analysis`
    - `bot_analysis`
  - Computes:
    - high-risk topics from cluster coherence and post volume
    - high-risk accounts from bot probabilities and behavioral patterns
    - attention weights
    - human-readable explainability summary
- **Key expressions or variables used:**
  - `$fromAI("narrative_analysis", ...)`
  - `$fromAI("bot_analysis", ...)`
- **Input and output connections:** Connected to **Bot Behavior Analyzer Agent** as `ai_tool`.
- **Version-specific requirements:** Type version 1.3.
- **Edge cases / failures:**
  - Assumes upstream tools return keys like `clusters` and `bot_signatures`.
  - If the bot agent invokes this tool before obtaining those results, output may be empty or partial.
  - Attention weights are rule-based, not learned.
- **Sub-workflow reference:** None.

---

## 2.4 Block: Manipulation Technique Classification

### Overview
This block identifies rhetorical and propaganda-style manipulation tactics in content. It uses a GPT-4o sub-agent and a Python taxonomy tool with keyword and regex-based detection rules.

### Nodes Involved
- Manipulation Technique Classifier Agent
- Manipulation Classifier Model
- Manipulation Taxonomy Tool

### Node Details

#### Manipulation Technique Classifier Agent
- **Type and role:** `@n8n/n8n-nodes-langchain.agentTool`; supervisor-callable sub-agent for manipulation tactic classification.
- **Configuration choices:**
  - Input comes from:
    `{{$fromAI('content', 'Social media post content to analyze for manipulation techniques')}}`
  - System message directs classification of:
    - emotional manipulation
    - logical fallacies
    - propaganda
    - disinformation tactics
- **Key expressions or variables used:**
  - `$fromAI('content', ...)`
- **Input and output connections:**
  - AI language model from **Manipulation Classifier Model**
  - AI tool from **Manipulation Taxonomy Tool**
  - Exposed upward to **Misinformation Detection Supervisor** as `ai_tool`
- **Version-specific requirements:** Type version 3.
- **Edge cases / failures:**
  - Depends on the supervisor passing suitable content, which may be a full dataset or a summary rather than a single post.
  - Tool detection is heuristic and pattern-based, not semantic.
- **Sub-workflow reference:** None.

#### Manipulation Classifier Model
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; GPT-4o model for manipulation analysis.
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0.1`
- **Key expressions or variables used:** None.
- **Input and output connections:** Connected to **Manipulation Technique Classifier Agent** via `ai_languageModel`.
- **Version-specific requirements:** Type version 1.3.
- **Edge cases / failures:**
  - OpenAI credential, access, or rate-limit issues.
- **Sub-workflow reference:** None.

#### Manipulation Taxonomy Tool
- **Type and role:** `@n8n/n8n-nodes-langchain.toolCode`; Python tool for rule-based manipulation detection.
- **Configuration choices:**
  - Expects plain text input via:
    `$fromAI("text", "Text content to analyze for manipulation techniques", "string")`
  - Scans for predefined patterns in categories:
    - emotional manipulation
    - logical fallacy
    - propaganda
    - disinformation tactic
  - Computes:
    - technique list
    - total count
    - manipulation score
    - average confidence
- **Key expressions or variables used:**
  - `$fromAI("text", ..., "string")`
- **Input and output connections:** Connected to **Manipulation Technique Classifier Agent** as `ai_tool`.
- **Version-specific requirements:** Type version 1.3.
- **Edge cases / failures:**
  - Keyword matching can overflag common rhetoric.
  - Regex for percentages and counts is broad and may classify neutral statistical statements as misleading.
  - If content is empty, it returns no techniques.
  - This tool returns a schema different from the final parser’s expected `narrative_analysis.manipulation_score`, so the supervisor must reconcile fields itself.
- **Sub-workflow reference:** None.

---

## 2.5 Block: Result Formatting and Persistence

### Overview
This block flattens the structured AI output into sheet-friendly fields and appends the record to Google Sheets. It preserves both summary columns and the full serialized assessment.

### Nodes Involved
- Format Results
- Store Risk Assessment

### Node Details

#### Format Results
- **Type and role:** `n8n-nodes-base.set`; reshapes the structured supervisor output into a compact record.
- **Configuration choices:**
  - Creates fields:
    - `timestamp` = current ISO time
    - `overall_risk_level`
    - `coordinated_narratives_count`
    - `bot_signatures_count`
    - `high_risk_topics`
    - `recommendations`
    - `full_analysis`
  - Uses nested references under `$json.output`, which implies the supervisor node returns parsed content in an `output` object.
- **Key expressions or variables used:**
  - `={{ $now.toISO() }}`
  - `={{ $json.output.overall_risk_level }}`
  - `={{ $json.output.narrative_analysis.coordinated_narratives.length }}`
  - `={{ $json.output.bot_analysis.bot_signatures.length }}`
  - `={{ $json.output.discourse_risk_heatmap.high_risk_topics.join(', ') }}`
  - `={{ $json.output.recommendations.join(' | ') }}`
  - `={{ JSON.stringify($json.output) }}`
- **Input and output connections:**
  - Main input from **Misinformation Detection Supervisor**
  - Main output to **Store Risk Assessment**
- **Version-specific requirements:** Type version 3.4.
- **Edge cases / failures:**
  - Any missing nested field will cause expression failures.
  - If arrays are undefined, `.length` or `.join()` will fail.
  - Consider defensive expressions if using with inconsistent LLM output.
- **Sub-workflow reference:** None.

#### Store Risk Assessment
- **Type and role:** `n8n-nodes-base.googleSheets`; appends the formatted result to a Google Sheet.
- **Configuration choices:**
  - Operation: `append`
  - Spreadsheet document ID: not filled in the JSON
  - Sheet name: not filled in the JSON
  - Uses Google Sheets OAuth2 credentials
- **Key expressions or variables used:** None in current config.
- **Input and output connections:** Main input from **Format Results**.
- **Version-specific requirements:** Type version 4.7.
- **Edge cases / failures:**
  - Will fail until `documentId` and `sheetName` are configured.
  - OAuth permission issues or expired credentials.
  - Column mismatch if the target sheet headers are not aligned with fields being appended.
  - Google API quota or network errors.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Start Analysis | n8n-nodes-base.manualTrigger | Manual workflow entry point |  | Misinformation Detection Supervisor | ## How It Works<br>This workflow automates misinformation and information manipulation detection using a coordinated multi-agent AI architecture. It is designed for trust and safety teams, media analysts, researchers, and platform moderators who need scalable, structured threat assessment. The pipeline begins when a trigger initiates content analysis. A central Misinformation Detection supervisor agent coordinates three specialised sub-agents: a Narrative Pattern Detector that identifies recurring disinformation themes via semantic clustering, a Bot Behaviour Analyser that detects coordinated inauthentic activity using propagation and temporal pattern tools, and a Manipulation Technique Classifier that maps content to known influence tactics using a risk heatmap and taxonomy tools. Each agent uses a dedicated AI model and memory. Results are passed to a structured output parser, formatted for readability, and appended to Google Sheets for ongoing risk tracking and audit. |
| Misinformation Detection Supervisor | @n8n/n8n-nodes-langchain.agent | Central AI orchestrator coordinating specialist agents | Start Analysis, Supervisor Model, Narrative Pattern Detector Agent, Bot Behavior Analyzer Agent, Manipulation Technique Classifier Agent, Risk Assessment Output Parser | Format Results | ## Misinformation Detection Agent<br>**What:** Supervises three sub-agents via chat memory and tool routing.<br>**Why:** Centralises orchestration for consistent, parallel analysis. |
| Supervisor Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | GPT-4o model backing the supervisor agent |  | Misinformation Detection Supervisor | ## Setup Steps<br>1. Connect OpenAI credentials to Supervisor, Narrative, Bot, and Manipulation classifier model nodes.<br>2. Configure Google Sheets credentials and set target spreadsheet ID in the Store Risk Assessment node.<br>3. Set memory buffer windows in each sub-agent's Memory node to match your analysis context length. |
| Risk Assessment Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces structured final JSON schema |  | Misinformation Detection Supervisor | ## Manipulation Technique Classifier Agent<br>**What:** Maps content to manipulation tactics with risk heatmap.<br>**Why:** Enables structured, taxonomy-driven threat scoring. |
| Narrative Pattern Detector Agent | @n8n/n8n-nodes-langchain.agentTool | Specialist agent for coordinated narrative detection | Narrative Detector Model, Semantic Clustering Tool | Misinformation Detection Supervisor | ## Narrative Pattern Detector Agent<br>**What:** Identifies disinformation narratives using semantic clustering.<br>**Why:** Surfaces recurring themes missed by keyword filters. |
| Narrative Detector Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | GPT-4o model for narrative sub-agent |  | Narrative Pattern Detector Agent | ## Narrative Pattern Detector Agent<br>**What:** Identifies disinformation narratives using semantic clustering.<br>**Why:** Surfaces recurring themes missed by keyword filters. |
| Semantic Clustering Tool | @n8n/n8n-nodes-langchain.toolCode | Python TF-IDF clustering tool for semantic grouping |  | Narrative Pattern Detector Agent | ## Narrative Pattern Detector Agent<br>**What:** Identifies disinformation narratives using semantic clustering.<br>**Why:** Surfaces recurring themes missed by keyword filters. |
| Bot Behavior Analyzer Agent | @n8n/n8n-nodes-langchain.agentTool | Specialist agent for bot signatures and propagation analysis | Bot Analyzer Model, Propagation Cascade Analyzer, Temporal Pattern Detector, Risk Heatmap Generator | Misinformation Detection Supervisor | ## Bot Behaviour Analyser Agent<br>**What:** Analyses propagation cascades and temporal patterns.<br>**Why:** Detects coordinated inauthentic amplification. |
| Bot Analyzer Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | GPT-4o model for bot-behavior sub-agent |  | Bot Behavior Analyzer Agent | ## Bot Behaviour Analyser Agent<br>**What:** Analyses propagation cascades and temporal patterns.<br>**Why:** Detects coordinated inauthentic amplification. |
| Propagation Cascade Analyzer | @n8n/n8n-nodes-langchain.toolCode | Python tool for repost/share cascade modeling |  | Bot Behavior Analyzer Agent | ## Bot Behaviour Analyser Agent<br>**What:** Analyses propagation cascades and temporal patterns.<br>**Why:** Detects coordinated inauthentic amplification. |
| Temporal Pattern Detector | @n8n/n8n-nodes-langchain.toolCode | Python tool for temporal bot-pattern scoring |  | Bot Behavior Analyzer Agent | ## Bot Behaviour Analyser Agent<br>**What:** Analyses propagation cascades and temporal patterns.<br>**Why:** Detects coordinated inauthentic amplification. |
| Risk Heatmap Generator | @n8n/n8n-nodes-langchain.toolCode | Combines narrative and bot results into explainable risk heatmap |  | Bot Behavior Analyzer Agent | ## Bot Behaviour Analyser Agent<br>**What:** Analyses propagation cascades and temporal patterns.<br>**Why:** Detects coordinated inauthentic amplification. |
| Manipulation Technique Classifier Agent | @n8n/n8n-nodes-langchain.agentTool | Specialist agent for rhetorical and propaganda tactic detection | Manipulation Classifier Model, Manipulation Taxonomy Tool | Misinformation Detection Supervisor | ## Manipulation Technique Classifier Agent<br>**What:** Maps content to manipulation tactics with risk heatmap.<br>**Why:** Enables structured, taxonomy-driven threat scoring. |
| Manipulation Classifier Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | GPT-4o model for manipulation analysis |  | Manipulation Technique Classifier Agent | ## Manipulation Technique Classifier Agent<br>**What:** Maps content to manipulation tactics with risk heatmap.<br>**Why:** Enables structured, taxonomy-driven threat scoring. |
| Manipulation Taxonomy Tool | @n8n/n8n-nodes-langchain.toolCode | Python taxonomy classifier for manipulation techniques |  | Manipulation Technique Classifier Agent | ## Manipulation Technique Classifier Agent<br>**What:** Maps content to manipulation tactics with risk heatmap.<br>**Why:** Enables structured, taxonomy-driven threat scoring. |
| Format Results | n8n-nodes-base.set | Flattens structured analysis into storage-ready fields | Misinformation Detection Supervisor | Store Risk Assessment | ## Format Results & Store Risk Assessment<br>**What:** Parses, formats, and appends findings to Google Sheets.<br>**Why:** Ensures auditable, structured reporting. |
| Store Risk Assessment | n8n-nodes-base.googleSheets | Appends analysis record to Google Sheets | Format Results |  | ## Format Results & Store Risk Assessment<br>**What:** Parses, formats, and appends findings to Google Sheets.<br>**Why:** Ensures auditable, structured reporting. |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas documentation for prerequisites and benefits |  |  | ## Prerequisites<br>- Google Sheets API credentials<br>- n8n instance (v1.0+)<br>- Access to propagation/temporal data APIs<br>- Google account with target Sheet pre-created<br>## Use Cases<br>- Platform trust and safety teams flagging viral misinformation campaigns<br>## Customisation<br>- Replace Google Sheets with a database or SIEM output<br>## Benefits<br>- Parallel multi-agent analysis cuts manual review time significantly |
| Sticky Note1 | n8n-nodes-base.stickyNote | Canvas setup guidance |  |  | ## Setup Steps<br>1. Connect OpenAI credentials to Supervisor, Narrative, Bot, and Manipulation classifier model nodes.<br>2. Configure Google Sheets credentials and set target spreadsheet ID in the Store Risk Assessment node.<br>3. Set memory buffer windows in each sub-agent's Memory node to match your analysis context length. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Canvas overview of workflow behavior |  |  | ## How It Works<br>This workflow automates misinformation and information manipulation detection using a coordinated multi-agent AI architecture. It is designed for trust and safety teams, media analysts, researchers, and platform moderators who need scalable, structured threat assessment. The pipeline begins when a trigger initiates content analysis. A central Misinformation Detection supervisor agent coordinates three specialised sub-agents: a Narrative Pattern Detector that identifies recurring disinformation themes via semantic clustering, a Bot Behaviour Analyser that detects coordinated inauthentic activity using propagation and temporal pattern tools, and a Manipulation Technique Classifier that maps content to known influence tactics using a risk heatmap and taxonomy tools. Each agent uses a dedicated AI model and memory. Results are passed to a structured output parser, formatted for readability, and appended to Google Sheets for ongoing risk tracking and audit. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Canvas note describing bot analysis area |  |  | ## Bot Behaviour Analyser Agent<br>**What:** Analyses propagation cascades and temporal patterns.<br>**Why:** Detects coordinated inauthentic amplification. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas note describing narrative analysis area |  |  | ## Narrative Pattern Detector Agent<br>**What:** Identifies disinformation narratives using semantic clustering.<br>**Why:** Surfaces recurring themes missed by keyword filters. |
| Sticky Note5 | n8n-nodes-base.stickyNote | Canvas note describing supervisor area |  |  | ## Misinformation Detection Agent<br>**What:** Supervises three sub-agents via chat memory and tool routing.<br>**Why:** Centralises orchestration for consistent, parallel analysis. |
| Sticky Note6 | n8n-nodes-base.stickyNote | Canvas note describing manipulation analysis area |  |  | ## Manipulation Technique Classifier Agent<br>**What:** Maps content to manipulation tactics with risk heatmap.<br>**Why:** Enables structured, taxonomy-driven threat scoring. |
| Sticky Note7 | n8n-nodes-base.stickyNote | Canvas note describing storage area |  |  | ## Format Results & Store Risk Assessment<br>**What:** Parses, formats, and appends findings to Google Sheets.<br>**Why:** Ensures auditable, structured reporting. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and give it a name similar to:
   - `Detect misinformation and manipulation risks using multi-agent AI analysis`

2. **Add a Manual Trigger node**
   - Node type: `Manual Trigger`
   - Name it: `Start Analysis`

3. **Add the supervisor model node**
   - Node type: `OpenAI Chat Model` from the LangChain nodes
   - Name it: `Supervisor Model`
   - Select OpenAI credentials
   - Model: `gpt-4o`
   - Temperature: `0.3`

4. **Add the main supervisor agent**
   - Node type: `AI Agent`
   - Name it: `Misinformation Detection Supervisor`
   - Set prompt mode to define/custom prompt
   - Set input text to:
     - `{{$json.posts}}`
   - Set max iterations to `5`
   - Set the system message to instruct the agent to coordinate:
     - the narrative detector
     - the bot behavior analyzer
     - the manipulation classifier
     - and synthesize findings into a unified discourse risk assessment
   - Enable structured output parser support

5. **Connect the trigger and supervisor**
   - `Start Analysis` → `Misinformation Detection Supervisor` via main connection
   - `Supervisor Model` → `Misinformation Detection Supervisor` via `ai_languageModel`

6. **Add the structured output parser**
   - Node type: `Structured Output Parser`
   - Name it: `Risk Assessment Output Parser`
   - Paste a JSON schema example defining:
     - `overall_risk_level` with enum `critical/high/moderate/low`
     - `narrative_analysis.coordinated_narratives`
     - `narrative_analysis.manipulation_score`
     - `bot_analysis.bot_signatures`
     - `bot_analysis.propagation_cascades`
     - `discourse_risk_heatmap`
     - `recommendations`
   - Connect it to the supervisor as `ai_outputParser`

7. **Create the narrative-analysis model**
   - Node type: `OpenAI Chat Model`
   - Name it: `Narrative Detector Model`
   - Credentials: same OpenAI account
   - Model: `gpt-4o`
   - Temperature: `0.2`

8. **Create the narrative-analysis agent tool**
   - Node type: `AI Agent Tool`
   - Name it: `Narrative Pattern Detector Agent`
   - Input text:
     - `{{$fromAI('post_dataset', 'Social media post dataset to analyze for narrative patterns')}}`
   - Add a system message instructing the agent to analyze coordinated narratives, manipulation techniques, and coherence indicators
   - Add a tool description stating it performs semantic clustering and returns narrative clusters and manipulation scores

9. **Connect the narrative model**
   - `Narrative Detector Model` → `Narrative Pattern Detector Agent` via `ai_languageModel`

10. **Create the semantic clustering code tool**
    - Node type: `Code Tool`
    - Name it: `Semantic Clustering Tool`
    - Language: `Python`
    - Paste the Python logic for:
      - tokenization
      - TF-IDF calculation
      - cosine similarity
      - threshold-based clustering
      - cluster theme extraction
    - Ensure it expects:
      - `$fromAI("posts", "Array of social media posts with content field", "object")`
    - Return JSON with `clusters` and `total_clusters`

11. **Connect the semantic clustering tool**
    - `Semantic Clustering Tool` → `Narrative Pattern Detector Agent` via `ai_tool`

12. **Expose the narrative agent to the supervisor**
    - `Narrative Pattern Detector Agent` → `Misinformation Detection Supervisor` via `ai_tool`

13. **Create the bot-analysis model**
    - Node type: `OpenAI Chat Model`
    - Name it: `Bot Analyzer Model`
    - Credentials: OpenAI
    - Model: `gpt-4o`
    - Temperature: `0.2`

14. **Create the bot-analysis agent tool**
    - Node type: `AI Agent Tool`
    - Name it: `Bot Behavior Analyzer Agent`
    - Input text:
      - `{{$fromAI('activity_dataset', 'User activity and propagation data to analyze for bot behavior')}}`
    - Add system instructions for:
      - propagation cascade analysis
      - temporal anomaly detection
      - bot signature scoring
      - risk heatmap generation

15. **Connect the bot model**
    - `Bot Analyzer Model` → `Bot Behavior Analyzer Agent` via `ai_languageModel`

16. **Create the propagation cascade code tool**
    - Node type: `Code Tool`
    - Name it: `Propagation Cascade Analyzer`
    - Language: `Python`
    - Paste code that:
      - parses timestamps
      - builds parent-child repost graph
      - performs BFS for depth
      - computes spread velocity
      - returns significant cascades
    - Ensure the tool reads:
      - `$fromAI("activity_data", "Array of posts with repost/share relationships and timestamps", "object")`

17. **Connect the propagation tool**
    - `Propagation Cascade Analyzer` → `Bot Behavior Analyzer Agent` via `ai_tool`

18. **Create the temporal pattern code tool**
    - Node type: `Code Tool`
    - Name it: `Temporal Pattern Detector`
    - Language: `Python`
    - Paste code that:
      - groups posts by user
      - computes intervals, variance, burst ratio, regularity, hour diversity
      - estimates `bot_probability`
      - returns suspicious bot signatures
    - Ensure it reads:
      - `$fromAI("user_activity", "Array of user posts with timestamps and user IDs", "object")`

19. **Connect the temporal tool**
    - `Temporal Pattern Detector` → `Bot Behavior Analyzer Agent` via `ai_tool`

20. **Create the risk heatmap code tool**
    - Node type: `Code Tool`
    - Name it: `Risk Heatmap Generator`
    - Language: `Python`
    - Paste code that:
      - combines narrative and bot analysis outputs
      - computes high-risk topics and accounts
      - calculates attention weights
      - builds explainability text
    - Ensure it expects:
      - `$fromAI("narrative_analysis", ...)`
      - `$fromAI("bot_analysis", ...)`

21. **Connect the risk heatmap tool**
    - `Risk Heatmap Generator` → `Bot Behavior Analyzer Agent` via `ai_tool`

22. **Expose the bot agent to the supervisor**
    - `Bot Behavior Analyzer Agent` → `Misinformation Detection Supervisor` via `ai_tool`

23. **Create the manipulation-analysis model**
    - Node type: `OpenAI Chat Model`
    - Name it: `Manipulation Classifier Model`
    - Credentials: OpenAI
    - Model: `gpt-4o`
    - Temperature: `0.1`

24. **Create the manipulation classifier agent tool**
    - Node type: `AI Agent Tool`
    - Name it: `Manipulation Technique Classifier Agent`
    - Input text:
      - `{{$fromAI('content', 'Social media post content to analyze for manipulation techniques')}}`
    - Add system instructions covering:
      - emotional manipulation
      - logical fallacies
      - propaganda techniques
      - disinformation tactics

25. **Connect the manipulation model**
    - `Manipulation Classifier Model` → `Manipulation Technique Classifier Agent` via `ai_languageModel`

26. **Create the manipulation taxonomy code tool**
    - Node type: `Code Tool`
    - Name it: `Manipulation Taxonomy Tool`
    - Language: `Python`
    - Paste code that:
      - classifies text with keyword lists and regex patterns
      - groups matches into categories
      - returns `techniques`, `total_techniques`, `manipulation_score`, `avg_confidence`
    - Ensure it expects:
      - `$fromAI("text", "Text content to analyze for manipulation techniques", "string")`

27. **Connect the manipulation taxonomy tool**
    - `Manipulation Taxonomy Tool` → `Manipulation Technique Classifier Agent` via `ai_tool`

28. **Expose the manipulation agent to the supervisor**
    - `Manipulation Technique Classifier Agent` → `Misinformation Detection Supervisor` via `ai_tool`

29. **Add a Set node for output formatting**
    - Node type: `Set`
    - Name it: `Format Results`
    - Create these fields:
      1. `timestamp` = `{{$now.toISO()}}`
      2. `overall_risk_level` = `{{$json.output.overall_risk_level}}`
      3. `coordinated_narratives_count` = `{{$json.output.narrative_analysis.coordinated_narratives.length}}`
      4. `bot_signatures_count` = `{{$json.output.bot_analysis.bot_signatures.length}}`
      5. `high_risk_topics` = `{{$json.output.discourse_risk_heatmap.high_risk_topics.join(', ')}}`
      6. `recommendations` = `{{$json.output.recommendations.join(' | ')}}`
      7. `full_analysis` = `{{JSON.stringify($json.output)}}`

30. **Connect the supervisor to formatting**
    - `Misinformation Detection Supervisor` → `Format Results`

31. **Add a Google Sheets node**
    - Node type: `Google Sheets`
    - Name it: `Store Risk Assessment`
    - Operation: `Append`
    - Select Google Sheets OAuth2 credentials
    - Set:
      - Spreadsheet document ID
      - Sheet name
    - Ensure the target sheet has compatible headers for the fields from `Format Results`

32. **Connect formatting to Google Sheets**
    - `Format Results` → `Store Risk Assessment`

33. **Prepare the input payload**
    - Since the workflow starts with a Manual Trigger, you need test data available at runtime.
    - The supervisor expects an item with a `posts` field.
    - In practice, you will usually add an upstream ingestion node later, or pin test data.
    - The dataset should ideally include, per post:
      - `content`
      - `id` or `post_id`
      - `timestamp` or `created_at`
      - `user_id` or `author_id`
      - `repost_of` or `parent_id` for propagation analysis

34. **Configure credentials**
    - **OpenAI**: one credential can be reused for all four model nodes.
    - **Google Sheets OAuth2**: authorize access to the destination spreadsheet.
    - Confirm the OpenAI account has access to `gpt-4o`.

35. **Recommended hardening before production**
    - Add an input-preparation node before the supervisor to normalize fields and guarantee `posts` exists.
    - Add error handling for missing arrays in the `Format Results` node.
    - Replace Manual Trigger with Webhook, Schedule, Google Sheets Trigger, database query, or API ingestion.
    - Consider replacing Google Sheets with a database, SIEM, or queue for higher-volume operations.

36. **Important implementation note**
    - The sticky note mentions memory nodes, but no memory nodes are actually present in this workflow JSON.
    - You do not need to create memory nodes to reproduce the workflow exactly as provided.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Prerequisites: Google Sheets API credentials, n8n instance (v1.0+), access to propagation/temporal data APIs, Google account with target Sheet pre-created | Environment/setup requirements |
| Use case: Platform trust and safety teams flagging viral misinformation campaigns | Operational context |
| Customisation: Replace Google Sheets with a database or SIEM output | Deployment option |
| Benefit: Parallel multi-agent analysis cuts manual review time significantly | Architecture benefit |
| Setup steps note says to connect OpenAI credentials to Supervisor, Narrative, Bot, and Manipulation classifier model nodes | Credential setup guidance |
| Setup steps note says to configure Google Sheets credentials and set target spreadsheet ID in the Store Risk Assessment node | Storage setup guidance |
| Setup steps note says to set memory buffer windows in each sub-agent's Memory node to match your analysis context length, but this specific workflow JSON contains no memory nodes | Consistency note |
| The workflow is inactive in the provided JSON (`active: false`) | Deployment status |
| Google Sheets node is incomplete in the provided JSON because both `documentId` and `sheetName` are blank | Required manual completion |
| The workflow relies on the incoming item containing `posts`, but the current Manual Trigger alone does not generate that payload | Input design constraint |
Screen and score job candidates with GPT-4o, Gmail and Google Calendar

https://n8nworkflows.xyz/workflows/screen-and-score-job-candidates-with-gpt-4o--gmail-and-google-calendar-13321


# Screen and score job candidates with GPT-4o, Gmail and Google Calendar

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title (given):** Screen and score job candidates with GPT-4o, Gmail and Google Calendar  
**Workflow name (JSON):** AI-powered candidate validation and trust assessment system

This workflow collects candidate applications via an n8n Form, extracts CV text, normalizes all inputs into a structured candidate profile, and runs a **multi-agent AI evaluation** (signals validation, CV verification, trust assessment, and candidate experience actions). An **Orchestrator Agent** synthesizes these results into a final decision. The workflow then **routes** to an approval or rejection email and **logs** the final outcome.

### 1.1 Input Reception & Configuration
- Candidate submits a form + CV upload.
- Workflow injects configurable constants (HR email, thresholds, company name).

### 1.2 Document Parsing & Data Normalization
- Extract text from the uploaded CV (PDF parsing).
- Map form fields + extracted CV text into a consistent JSON object.

### 1.3 Multi-Agent AI Screening (LangChain in n8n)
- Orchestrator (GPT-4o) calls specialized agent tools (GPT-4o-mini).
- Output parsers enforce structured JSON results per agent and for the orchestrator’s final response.
- Experience Agent can send Gmail and create Calendar events (tooling is available to the agent).

### 1.4 Decision Routing, Candidate Communication & Logging
- If overallStatus is **Approved** → send approval email.
- Else → send rejection email.
- Log result to console via Code node (not to Sheets/Calendar, despite sticky note mention).

---

## 2. Block-by-Block Analysis

### Block A — Application Intake & Global Settings
**Overview:** Captures candidate submission and sets workflow-wide configuration values used downstream (emails, scoring thresholds, company identity).  
**Nodes involved:** Candidate Application Form, Workflow Configuration

#### Node: Candidate Application Form
- **Type / role:** `formTrigger` — entry point; renders an n8n-hosted form and outputs submitted fields + uploaded file binary.
- **Configuration (interpreted):**
  - Form title/description: “Candidate Application Form…”
  - Fields include: Full Name, Email (email type), Phone, Position, Cover Letter (textarea), CV/Resume (file; required; accept .pdf/.doc/.docx), Years of Experience, LinkedIn URL, Previous Employers, Highest Education.
- **Inputs / outputs:**
  - **Output →** Workflow Configuration (main).
- **Key data shapes:**
  - Form outputs typically include human-readable labels as keys (e.g., `"Full Name"`, `"Email Address"`) and uploaded file data in binary.
- **Edge cases / failures:**
  - File upload missing (required) blocks submission.
  - CV file types include .doc/.docx, but downstream extraction node is configured for **PDF** only (see Block B).
- **Version notes:** typeVersion 2.5.

#### Node: Workflow Configuration
- **Type / role:** `set` — inject constants and keep incoming fields.
- **Configuration:**
  - Adds:
    - `hrEmail` (placeholder)
    - `minExperienceYears` = 2
    - `trustScoreThreshold` = 70
    - `validationScoreThreshold` = 75
    - `companyName` (placeholder)
  - “Include other fields” enabled → original form payload continues.
- **Inputs / outputs:**
  - **Input:** Candidate Application Form
  - **Output →** Extract CV Content
- **Edge cases / failures:**
  - Placeholders must be replaced; otherwise email sending will fail (invalid from address/company branding).
- **Version notes:** typeVersion 3.4.

**Sticky note context applied to this block:**
- **How It Works:** end-to-end candidate evaluation automation and routing; mentions Calendar/Sheets logging (Sheets not implemented).
- **Setup Steps:** connect trigger, configure extraction, OpenAI keys, prompts, Gmail + Calendar credentials.
- **Prerequisites / Use Cases / Customization / Benefits:** ATS/webhook access, OpenAI, role-specific prompt tuning, threshold tuning.

---

### Block B — CV Text Extraction & Candidate Profile Preparation
**Overview:** Extracts CV text (PDF) and maps all form fields + extracted CV into a normalized JSON schema used by the AI agents.  
**Nodes involved:** Extract CV Content, Prepare Candidate Data

#### Node: Extract CV Content
- **Type / role:** `extractFromFile` — document text extraction.
- **Configuration:**
  - Operation: **pdf**
  - Option: `joinPages: true` → merges multi-page text.
- **Inputs / outputs:**
  - **Input:** Workflow Configuration output (must include binary file content).
  - **Output →** Prepare Candidate Data
- **Key variables:**
  - Extracted text is placed into `$json.data` (then referenced later).
- **Edge cases / failures:**
  - If candidate uploads .doc/.docx, this node (PDF mode) may fail or output empty/invalid text.
  - Encrypted/scanned PDFs may extract poorly.
  - Large PDFs can cause timeouts or truncated extraction.
- **Version notes:** typeVersion 1.1.

#### Node: Prepare Candidate Data
- **Type / role:** `set` — normalize fields to consistent key names.
- **Configuration:**
  - Creates:
    - `candidateName = {{$json['Full Name']}}`
    - `candidateEmail = {{$json['Email Address']}}`
    - `candidatePhone = {{$json['Phone Number']}}`
    - `positionApplied = {{$json['Position Applied For']}}`
    - `yearsExperience = {{$json['Years of Experience']}}` (stored as string)
    - `linkedinProfile = {{$json['LinkedIn Profile URL']}}`
    - `coverLetter = {{$json['Cover Letter']}}`
    - `previousEmployers = {{$json['Previous Employers']}}`
    - `educationLevel = {{$json['Highest Education Level']}}`
    - `cvContent = {{$json.data}}` (extracted CV text)
  - Keeps original fields (`includeOtherFields: true`).
- **Inputs / outputs:**
  - **Input:** Extract CV Content
  - **Output →** Orchestrator Agent
- **Edge cases / failures:**
  - If form labels change, the bracketed label lookups (e.g., `['Full Name']`) break.
  - `yearsExperience` is a string; numeric comparisons require casting if used later.
- **Version notes:** typeVersion 3.4.

**Sticky note context applied to this block:**
- **Application Processing:** standardizes applications into consistent data for AI analysis.

---

### Block C — Multi-Agent AI Screening & Orchestration
**Overview:** A GPT-4o Orchestrator coordinates multiple specialized GPT-4o-mini “agent tools”, each returning structured results enforced by output parsers. The Orchestrator synthesizes a final structured decision.  
**Nodes involved:** Orchestrator Agent, Candidate Signal Agent Tool, CV Verification Agent Tool, Trust Assessment Agent Tool, Experience Agent Tool, all OpenAI Model nodes, all Output Parser nodes, Gmail Tool, Google Calendar Tool

#### Node: Orchestrator Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — central reasoning + tool-calling controller.
- **Configuration:**
  - User text: `Candidate Data: {{ JSON.stringify($json) }}`
  - System message: instructs orchestrator to call the 4 tools, synthesize, and return structured assessment.
  - Output parser enabled (`hasOutputParser: true`) → must match Orchestrator Output Parser schema.
- **Inputs / outputs:**
  - **Input:** Prepare Candidate Data (normalized candidate JSON)
  - **AI connections:**
    - Language model: OpenAI Model - Orchestrator
    - Tools available: Candidate Signal Agent Tool, CV Verification Agent Tool, Trust Assessment Agent Tool, Experience Agent Tool
    - Output parser: Orchestrator Output Parser
  - **Main output →** Check Validation Result
- **Edge cases / failures:**
  - Tool calling may omit needed fields unless prompts are robust.
  - If output parser schema isn’t met, the node errors (structured parsing failure).
  - Token limits: candidate CV text can be large; risk of truncation or cost spikes.
- **Version notes:** typeVersion 3.1.

#### Node: OpenAI Model - Orchestrator
- **Type / role:** `lmChatOpenAi` — GPT-4o model backing the orchestrator.
- **Configuration:**
  - Model: `gpt-4o`
  - Max tokens: 3000; Temperature: 0.3
- **Connections:**
  - **AI language model →** Orchestrator Agent
- **Edge cases:**
  - API quota/auth errors; model availability; latency.
- **Version notes:** typeVersion 1.3.

#### Node: Candidate Signal Agent Tool
- **Type / role:** `agentTool` — validates application data quality/completeness and alignment.
- **Configuration:**
  - Text: `Validate this candidate data: {{ $fromAI("candidateData", ..., "json") }}`
  - System message: score 0–100; return structured validation results.
  - Output parser enabled.
- **Connections:**
  - Model: OpenAI Model - Signal Agent
  - Output parser: Signal Agent Output Parser
  - Tool made available to: Orchestrator Agent (ai_tool)
- **Edge cases:**
  - If orchestrator doesn’t supply `candidateData` in expected shape, tool prompt becomes incomplete.
  - Parser failure if the agent returns non-JSON or missing required keys.
- **Version notes:** typeVersion 3.

#### Node: OpenAI Model - Signal Agent
- **Type / role:** `lmChatOpenAi` — GPT-4o-mini for signal validation.
- **Config:** `gpt-4o-mini`, maxTokens 2000, temperature 0.2
- **Connections:** AI language model → Candidate Signal Agent Tool
- **Version notes:** 1.3.

#### Node: Signal Agent Output Parser
- **Type / role:** `outputParserStructured` — enforces schema:
  - Required: `validationScore`, `status`, `completeness`
  - Optional arrays: `inconsistencies`, `missingInfo`, plus `findings`
- **Connections:** AI output parser → Candidate Signal Agent Tool
- **Failure mode:** strict schema mismatch → node/tool fails.

#### Node: CV Verification Agent Tool
- **Type / role:** `agentTool` — evaluates CV authenticity/consistency and flags timeline gaps.
- **Configuration:**
  - Text expects: `$fromAI("cvData", ..., "json")`
  - Output parser enabled.
- **Connections:**
  - Model: OpenAI Model - CV Agent
  - Output parser: CV Agent Output Parser
  - Tool available to: Orchestrator Agent
- **Edge cases:**
  - CV content may be empty (bad extraction), causing low-quality analysis or hallucinated assumptions.
- **Version notes:** typeVersion 3.

#### Node: OpenAI Model - CV Agent
- **Type / role:** `lmChatOpenAi` — GPT-4o-mini.
- **Config:** maxTokens 2000, temperature 0.2
- **Connections:** AI language model → CV Verification Agent Tool

#### Node: CV Agent Output Parser
- **Type / role:** structured schema enforcing:
  - Required: `authenticityScore`, `consistencyStatus`, `analysis`
  - Optional: `timelineGaps`, `inconsistencies`, `concerns`, `professionalismScore`
- **Connections:** AI output parser → CV Verification Agent Tool

#### Node: Trust Assessment Agent Tool
- **Type / role:** `agentTool` — evaluates candidate trustworthiness from provided signals (including LinkedIn URL quality, consistency).
- **Configuration:**
  - Text expects: `$fromAI("candidateProfile", ..., "json")`
  - Output parser enabled.
- **Connections:**
  - Model: OpenAI Model - Trust Agent
  - Output parser: Trust Agent Output Parser
  - Tool available to: Orchestrator Agent
- **Edge cases:**
  - No real web browsing is performed; LinkedIn “quality” is inferred only from given URL/text, not fetched content.
- **Version notes:** typeVersion 3.

#### Node: OpenAI Model - Trust Agent
- **Type / role:** `lmChatOpenAi` — GPT-4o-mini.
- **Config:** maxTokens 2000, temperature 0.2
- **Connections:** AI language model → Trust Assessment Agent Tool

#### Node: Trust Agent Output Parser
- **Type / role:** schema enforcing:
  - Required: `trustScore`, `reliabilityLevel`, `assessment`
  - Optional: `transparencyScore`, `consistencyScore`, arrays `reliabilityIndicators`, `riskFactors`
- **Connections:** AI output parser → Trust Assessment Agent Tool

#### Node: Experience Agent Tool
- **Type / role:** `agentTool` — executes candidate experience actions using Gmail/Calendar tools (send email, schedule interview, etc.).
- **Configuration:**
  - Text expects: `$fromAI("experienceAction", ..., "json")`
  - System message instructs using Gmail Tool + Google Calendar Tool.
- **Connections:**
  - Model: OpenAI Model - Experience Agent
  - Tools: Gmail Tool, Google Calendar Tool (ai_tool connections)
  - Tool available to: Orchestrator Agent
- **Edge cases:**
  - The overall workflow also sends approval/rejection via separate `emailSend` nodes; this creates potential **duplicate communications** if the Orchestrator triggers Experience Agent emails too.
  - Calendar event creation needs valid ISO timestamps; agent may produce invalid format.
- **Version notes:** typeVersion 3.

#### Node: OpenAI Model - Experience Agent
- **Type / role:** `lmChatOpenAi` — GPT-4o-mini with slightly higher temperature.
- **Config:** maxTokens 1500, temperature 0.4
- **Connections:** AI language model → Experience Agent Tool

#### Node: Gmail Tool
- **Type / role:** `gmailTool` — callable by AI agent to send emails.
- **Configuration (AI-filled):**
  - `sendTo` = `$fromAI('recipientEmail', ...)`
  - `subject` = `$fromAI('emailSubject', ...)`
  - `message` (HTML) = `$fromAI('emailBody', ...)`
- **Connections:** AI tool → Experience Agent Tool
- **Edge cases / failures:**
  - OAuth2 scope/consent issues; sending limits; invalid recipient.
  - AI may generate malformed HTML.
- **Version notes:** typeVersion 2.2.

#### Node: Google Calendar Tool
- **Type / role:** `googleCalendarTool` — callable by AI to create events.
- **Configuration (AI-filled):**
  - `start`, `end` ISO strings from AI
  - `calendarId` from AI (defaults to “primary” via the `fromAI` default)
  - `summary`, `description` from AI
- **Connections:** AI tool → Experience Agent Tool
- **Edge cases / failures:**
  - Invalid ISO timestamps/timezones; missing permissions; calendar not found.
- **Version notes:** typeVersion 1.3.

#### Node: Orchestrator Output Parser
- **Type / role:** structured parser enforcing the orchestrator’s final JSON:
  - Required: `overallStatus`, `validationScore`, `trustScore`, `recommendations`
  - Also supports: `cvAuthenticityScore`, `signalValidationScore`, `redFlags`, `strengths`, `nextActions`
- **Connections:** AI output parser → Orchestrator Agent
- **Failure mode:** orchestrator output not matching schema stops workflow.

**Sticky note context applied to this block:**
- **Multi-Agent Screening:** “Four specialized AI agents…”; mentions Calendar + Sheets storage (Sheets not present in actual nodes).

---

### Block D — Decision Routing, Email Response & Logging
**Overview:** Routes based on orchestrator decision, sends candidate-facing email, then logs results as a structured record.  
**Nodes involved:** Check Validation Result, Send Approval Email, Send Rejection Email, Log Results

#### Node: Check Validation Result
- **Type / role:** `if` — branching logic.
- **Condition:** `$json.overallStatus` equals `"Approved"` (case-insensitive setting but equality value is exact).
- **Connections:**
  - **True →** Send Approval Email
  - **False →** Send Rejection Email
- **Edge cases:**
  - If orchestrator returns `"approved"` or `"APPROVED"` it should still pass due to case-insensitive option (as configured).
  - If orchestrator returns other statuses like “Review Required”, it goes to rejection branch (no dedicated review path).
- **Version notes:** typeVersion 2.3.

#### Node: Send Approval Email
- **Type / role:** `emailSend` — sends approval decision to candidate.
- **Configuration (key expressions):**
  - `toEmail`: `$('Prepare Candidate Data').first().json.candidateEmail`
  - `fromEmail`: `$('Workflow Configuration').first().json.hrEmail`
  - Subject includes position + company name.
  - HTML body includes scores and renders `nextActions` list:
    - `{{ $json.nextActions.map(action => `<li>${action}</li>`).join("") }}`
- **Connections:** **Output →** Log Results
- **Edge cases / failures:**
  - If `nextActions` is missing/not array, `.map` throws expression error.
  - SMTP/Email node requires proper email configuration in n8n; `fromEmail` must be authorized.
- **Version notes:** typeVersion 2.1.

#### Node: Send Rejection Email
- **Type / role:** `emailSend` — sends rejection decision.
- **Configuration:**
  - Similar addressing/subject strategy as approval.
  - Conditionally includes “Areas for Improvement” if `redFlags` exists and has length:
    - `{{ $json.redFlags && $json.redFlags.length > 0 ? `<h3>...` : "" }}`
- **Connections:** **Output →** Log Results
- **Edge cases / failures:**
  - If `redFlags` is not an array but exists, `.length` may behave unexpectedly.
- **Version notes:** typeVersion 2.1.

#### Node: Log Results
- **Type / role:** `code` — builds a log entry and prints it; returns it as workflow output.
- **Logic (interpreted):**
  - Pulls candidate data from **Prepare Candidate Data**.
  - Pulls orchestrator results from `$input.first().json`.
  - Constructs `logEntry` with timestamp, candidate identifiers, all scores, lists, and `emailSent: true`.
  - `console.log(...)` and returns `[ { json: logEntry } ]`.
- **Connections:** terminal node (no further outputs).
- **Edge cases / failures:**
  - If either referenced node has no items, `.first()` can throw.
  - “emailSent: true” is always set even if the email node failed upstream (but in practice Log Results runs only after email node success).
- **Version notes:** typeVersion 2.

**Sticky note context applied to this block:**
- **Automated Response:** routes approvals/rejections and logs outcomes (mentions calendar/sheets logging; actual logging is Code node only).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Candidate Application Form | n8n-nodes-base.formTrigger | Entry form intake (candidate submission + file upload) | — | Workflow Configuration | ## How It Works<br>This workflow automates end-to-end candidate evaluation… logging all outcomes to Google Calendar and Sheets…; ## Setup Steps…; ## Prerequisites…; ## Application Processing… |
| Workflow Configuration | n8n-nodes-base.set | Inject constants (HR email, thresholds, company name) | Candidate Application Form | Extract CV Content | ## How It Works…; ## Setup Steps…; ## Prerequisites…; ## Application Processing… |
| Extract CV Content | n8n-nodes-base.extractFromFile | Extract text from CV (PDF) | Workflow Configuration | Prepare Candidate Data | ## Setup Steps…; ## Application Processing… |
| Prepare Candidate Data | n8n-nodes-base.set | Normalize fields + attach cvContent | Extract CV Content | Orchestrator Agent | ## Application Processing…; ## Multi-Agent Screening… |
| Orchestrator Agent | @n8n/n8n-nodes-langchain.agent | Coordinates agent tools; produces final decision JSON | Prepare Candidate Data | Check Validation Result | ## Multi-Agent Screening… |
| Candidate Signal Agent Tool | @n8n/n8n-nodes-langchain.agentTool | Validates completeness/consistency; produces validation score | (Tool called by Orchestrator Agent) | (Returns to Orchestrator Agent) | ## Multi-Agent Screening… |
| CV Verification Agent Tool | @n8n/n8n-nodes-langchain.agentTool | Checks CV authenticity/consistency; flags concerns | (Tool called by Orchestrator Agent) | (Returns to Orchestrator Agent) | ## Multi-Agent Screening… |
| Trust Assessment Agent Tool | @n8n/n8n-nodes-langchain.agentTool | Trustworthiness evaluation; trust score + risks | (Tool called by Orchestrator Agent) | (Returns to Orchestrator Agent) | ## Multi-Agent Screening… |
| Experience Agent Tool | @n8n/n8n-nodes-langchain.agentTool | Candidate comms + scheduling via Gmail/Calendar tools | (Tool called by Orchestrator Agent) | (Returns to Orchestrator Agent) | ## Multi-Agent Screening… |
| OpenAI Model - Orchestrator | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM backend for orchestrator (gpt-4o) | — | Orchestrator Agent (AI model link) | ## Multi-Agent Screening… |
| OpenAI Model - Signal Agent | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM backend for signal agent (gpt-4o-mini) | — | Candidate Signal Agent Tool (AI model link) | ## Multi-Agent Screening… |
| OpenAI Model - CV Agent | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM backend for CV agent (gpt-4o-mini) | — | CV Verification Agent Tool (AI model link) | ## Multi-Agent Screening… |
| OpenAI Model - Trust Agent | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM backend for trust agent (gpt-4o-mini) | — | Trust Assessment Agent Tool (AI model link) | ## Multi-Agent Screening… |
| OpenAI Model - Experience Agent | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM backend for experience agent (gpt-4o-mini) | — | Experience Agent Tool (AI model link) | ## Multi-Agent Screening… |
| Orchestrator Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces final orchestrator JSON schema | — | Orchestrator Agent (AI parser link) | ## Multi-Agent Screening… |
| Signal Agent Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces signal agent JSON schema | — | Candidate Signal Agent Tool (AI parser link) | ## Multi-Agent Screening… |
| CV Agent Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces CV agent JSON schema | — | CV Verification Agent Tool (AI parser link) | ## Multi-Agent Screening… |
| Trust Agent Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces trust agent JSON schema | — | Trust Assessment Agent Tool (AI parser link) | ## Multi-Agent Screening… |
| Gmail Tool | n8n-nodes-base.gmailTool | AI-callable Gmail sending tool | — | Experience Agent Tool (AI tool link) | ## Multi-Agent Screening… |
| Google Calendar Tool | n8n-nodes-base.googleCalendarTool | AI-callable Calendar event creation tool | — | Experience Agent Tool (AI tool link) | ## Multi-Agent Screening… |
| Check Validation Result | n8n-nodes-base.if | Branch on `overallStatus == Approved` | Orchestrator Agent | Send Approval Email; Send Rejection Email | ## Automated Response… |
| Send Approval Email | n8n-nodes-base.emailSend | Candidate approval email (templated HTML) | Check Validation Result (true) | Log Results | ## Automated Response… |
| Send Rejection Email | n8n-nodes-base.emailSend | Candidate rejection email (templated HTML) | Check Validation Result (false) | Log Results | ## Automated Response… |
| Log Results | n8n-nodes-base.code | Build structured logEntry + console.log | Send Approval Email / Send Rejection Email | — | ## Automated Response… |
| Sticky Note | n8n-nodes-base.stickyNote | Comment container | — | — | ## How It Works… |
| Sticky Note1 | n8n-nodes-base.stickyNote | Comment container | — | — | ## Setup Steps… |
| Sticky Note2 | n8n-nodes-base.stickyNote | Comment container | — | — | ## Prerequisites… |
| Sticky Note3 | n8n-nodes-base.stickyNote | Comment container | — | — | ## Application Processing… |
| Sticky Note4 | n8n-nodes-base.stickyNote | Comment container | — | — | ## Multi-Agent Screening… |
| Sticky Note5 | n8n-nodes-base.stickyNote | Comment container | — | — | ## Automated Response… |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Form Trigger**
   - Add node: **Form Trigger**
   - Title: “Candidate Application Form”
   - Add fields:
     - Full Name (required)
     - Email Address (type email, required)
     - Phone Number (required)
     - Position Applied For (required)
     - Cover Letter (textarea, optional)
     - CV/Resume (file, required, single file, accept .pdf/.doc/.docx)
     - Years of Experience (required)
     - LinkedIn Profile URL (optional)
     - Previous Employers (textarea, optional)
     - Highest Education Level (optional)

2) **Add configuration constants**
   - Add node: **Set** (name: Workflow Configuration)
   - Add fields:
     - `hrEmail` (string) → set to your HR sender address
     - `minExperienceYears` (number) → 2
     - `trustScoreThreshold` (number) → 70
     - `validationScoreThreshold` (number) → 75
     - `companyName` (string) → your company
   - Enable **Include Other Fields**
   - Connect: **Form Trigger → Workflow Configuration**

3) **Extract CV text**
   - Add node: **Extract From File** (name: Extract CV Content)
   - Operation: **PDF**
   - Option: **Join pages = true**
   - Connect: **Workflow Configuration → Extract CV Content**
   - (Recommended fix) If you truly accept .doc/.docx, add a file-type router and dedicated extraction path(s), or restrict accept types to PDF only.

4) **Normalize candidate data**
   - Add node: **Set** (name: Prepare Candidate Data)
   - Enable **Include Other Fields**
   - Map:
     - `candidateName` = `{{$json["Full Name"]}}`
     - `candidateEmail` = `{{$json["Email Address"]}}`
     - `candidatePhone` = `{{$json["Phone Number"]}}`
     - `positionApplied` = `{{$json["Position Applied For"]}}`
     - `yearsExperience` = `{{$json["Years of Experience"]}}`
     - `linkedinProfile` = `{{$json["LinkedIn Profile URL"]}}`
     - `coverLetter` = `{{$json["Cover Letter"]}}`
     - `previousEmployers` = `{{$json["Previous Employers"]}}`
     - `educationLevel` = `{{$json["Highest Education Level"]}}`
     - `cvContent` = `{{$json.data}}`
   - Connect: **Extract CV Content → Prepare Candidate Data**

5) **Create OpenAI credentials**
   - In n8n Credentials: add **OpenAI API** credential.
   - Ensure it has access to `gpt-4o` and `gpt-4o-mini`.

6) **Add Orchestrator LLM**
   - Add node: **OpenAI Chat Model** (LangChain) (name: OpenAI Model - Orchestrator)
   - Model: `gpt-4o`, max tokens 3000, temperature 0.3
   - Select OpenAI credentials.

7) **Add Agent Tools + their LLMs + Output Parsers**
   - For each specialized agent (Signal/CV/Trust/Experience):
     - Add **OpenAI Chat Model** node (gpt-4o-mini) with the token/temp values:
       - Signal: 2000 / 0.2
       - CV: 2000 / 0.2
       - Trust: 2000 / 0.2
       - Experience: 1500 / 0.4
     - Add **Agent Tool** node with the corresponding system message (role instructions) and enable structured output parsing where present.
     - Add **Structured Output Parser** node and paste the schema (manual JSON schema) exactly as in the workflow for that tool.
     - Connect each:
       - Model (ai_languageModel) → Agent Tool
       - Output Parser (ai_outputParser) → Agent Tool

8) **Add Gmail + Google Calendar AI tools**
   - Create credentials:
     - **Gmail OAuth2** credential (authorize sending)
     - **Google Calendar OAuth2** credential (authorize event creation)
   - Add node: **Gmail Tool**
     - Map fields using `$fromAI(...)`: recipientEmail, emailSubject, emailBody (HTML)
   - Add node: **Google Calendar Tool**
     - start/end from `$fromAI` (ISO strings)
     - calendarId from `$fromAI` defaulting to `primary`
     - summary/description from `$fromAI`
   - Connect both (ai_tool) → **Experience Agent Tool**

9) **Add Orchestrator Output Parser**
   - Add **Structured Output Parser** (name: Orchestrator Output Parser)
   - Schema includes required `overallStatus`, `validationScore`, `trustScore`, `recommendations`, plus optional fields.

10) **Add Orchestrator Agent**
   - Add node: **AI Agent (LangChain)** (name: Orchestrator Agent)
   - User text: `Candidate Data: {{ JSON.stringify($json) }}`
   - System message: instruct calling all 4 tools and synthesizing final decision.
   - Attach:
     - ai_languageModel: **OpenAI Model - Orchestrator**
     - ai_tool: connect the 4 Agent Tools to Orchestrator (Candidate Signal / CV Verification / Trust Assessment / Experience Agent)
     - ai_outputParser: **Orchestrator Output Parser**
   - Connect: **Prepare Candidate Data → Orchestrator Agent**

11) **Decision branching**
   - Add node: **IF** (name: Check Validation Result)
   - Condition: `{{$json.overallStatus}}` equals `Approved`
   - Connect: **Orchestrator Agent → Check Validation Result**

12) **Send outcome emails (non-AI email nodes)**
   - Add node: **Send Email** (name: Send Approval Email)
     - To: `{{$('Prepare Candidate Data').first().json.candidateEmail}}`
     - From: `{{$('Workflow Configuration').first().json.hrEmail}}`
     - Subject/body as per your template; if you keep the provided template, ensure `nextActions` is always an array.
   - Add node: **Send Email** (name: Send Rejection Email)
     - To/From similarly
     - Conditional redFlags rendering
   - Connect:
     - IF true → Send Approval Email
     - IF false → Send Rejection Email
   - Configure the Email Send node’s transport in n8n (SMTP or provider), as required in your environment.

13) **Logging**
   - Add node: **Code** (name: Log Results)
   - Build a `logEntry` from:
     - `$('Prepare Candidate Data').first().json`
     - `$input.first().json` (orchestrator result)
   - Connect:
     - Send Approval Email → Log Results
     - Send Rejection Email → Log Results

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Sticky notes claim results are logged to Google Calendar and Google Sheets. In the actual workflow, only a **Google Calendar Tool** exists (AI-callable) and final logging is done via **Code node console.log**; there is no Google Sheets node. | Align expectations / consider adding a Google Sheets node for persistent audit logging. |
| The form accepts `.doc/.docx` but the extraction step is configured for **PDF** parsing only. | Consider restricting uploads to PDF or adding DOC/DOCX parsing paths. |
| Experience Agent can send Gmail/Calendar actions, while the workflow separately sends approval/rejection via Email Send nodes. | Risk of duplicate communications unless the orchestrator is constrained not to email candidates directly. |
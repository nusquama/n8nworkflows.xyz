Automate corporate tax filing for SMEs with GPT-5.2-Pro, Gmail, and Google Drive

https://n8nworkflows.xyz/workflows/automate-corporate-tax-filing-for-smes-with-gpt-5-2-pro--gmail--and-google-drive-12031


# Automate corporate tax filing for SMEs with GPT-5.2-Pro, Gmail, and Google Drive

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name:** *GPT-5.2-Pro SME Corporate Tax Filing and Compliance Automation*  
**User-facing title:** *Automate corporate tax filing for SMEs with GPT-5.2-Pro, Gmail, and Google Drive*

**Purpose:**  
Automates a monthly corporate tax pre-filing cycle for an SME: fetches monthly financial data from an accounting API, has an AI agent compute tax figures with explicit calculator-based arithmetic, formats a pre-filing report, has a second AI agent email the report to a tax agent (Gmail) and archive it (Google Drive), then optionally applies feedback/corrections and recalculates before finalizing submission metadata.

**Primary use cases:**
- Accounting/tax teams coordinating monthly close and pre-filing review.
- Firms handling multiple clients where a human tax agent reviews AI-prepared figures.
- Compliance-oriented organizations that must archive calculation artifacts.

### 1.1 Scheduling & Configuration
A monthly schedule triggers the workflow and sets operational parameters (API URL/key, tax agent email, Drive folder, tax rate, deductions).

### 1.2 Financial Data Retrieval
An HTTP request pulls accounting/financial statement data using the configured endpoint and bearer token.

### 1.3 Tax Calculation (AI + Structured Output)
A LangChain Agent (GPT-5.2-Pro) calculates gross profit, net profit, taxable income, liability, and effective rate, using a Calculator tool for arithmetic and a structured output parser for reliable JSON output.

### 1.4 Report Formatting
A Set node produces a human-readable summary and metadata (title/date/company).

### 1.5 Submission Coordination (AI + Gmail/Drive Tools)
A second LangChain Agent (GPT-5.2-Pro) uses:
- **Gmail tool** to email the pre-filing report to the tax agent,
- **Google Drive tool** to archive the JSON report to a compliance vault folder.

### 1.6 Feedback Handling & Finalization
An IF node routes execution:
- If feedback is received, a Code node merges corrections into the financial data and reruns the tax calculation block.
- Otherwise (or in parallel path as currently wired), a Set node stamps “Finalized” submission metadata.

> Important structural note: The workflow contains a “feedback loop” back into the Tax Calculation Agent. However, there is **no node that actually ingests feedback** (e.g., Gmail “watch inbox”, webhook form, or manual trigger). As-is, `feedbackReceived` must already exist in the incoming JSON for the IF to work.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduling & Workflow Configuration
**Overview:** Triggers monthly and defines runtime configuration values used by later nodes (API access, recipients, tax constants, Drive folder).  
**Nodes involved:**  
- Monthly Tax Filing Schedule  
- Workflow Configuration  

#### Node: Monthly Tax Filing Schedule
- **Type / role:** `scheduleTrigger` — time-based entry point.
- **Configuration (interpreted):** Runs on a monthly interval, triggering at **09:00** (server/workflow timezone).
- **Outputs:** Connects to **Workflow Configuration** (main).
- **Edge cases / failures:**
  - Timezone differences can cause unexpected run times.
  - Missed executions if n8n is down (depends on instance behavior and scheduling reliability).

#### Node: Workflow Configuration
- **Type / role:** `set` — central configuration injection.
- **Configuration choices:**
  - Sets these fields (placeholders indicate you must replace them):
    - `accountingApiUrl` (string, placeholder)
    - `accountingApiKey` (string, placeholder)
    - `taxAgentEmail` (string, placeholder)
    - `complianceVaultFolderId` (string, placeholder)
    - `taxRate` (number, default **0.21**)
    - `standardDeductions` (number, default **5000**)
  - **Include other fields:** enabled (keeps upstream data).
- **Key expressions/variables:** none; literal values except placeholders.
- **Input/Output connections:** Receives from Schedule; outputs to **Fetch Financial Data from Accounting Software**.
- **Edge cases / failures:**
  - Leaving placeholder values will break downstream HTTP/Gmail/Drive operations.
  - Incorrect Drive folder ID causes archive failure.
  - Wrong tax rate/deductions cause systematic miscalculation.

---

### Block 2 — Fetch Financial Data
**Overview:** Retrieves the monthly financial dataset from the accounting system via HTTP request, using the configured API endpoint and bearer token.  
**Nodes involved:**  
- Fetch Financial Data from Accounting Software  

#### Node: Fetch Financial Data from Accounting Software
- **Type / role:** `httpRequest` — pulls accounting data.
- **Configuration choices:**
  - **URL:** from configuration: `$('Workflow Configuration').first().json.accountingApiUrl`
  - **Auth header:** `Authorization: Bearer <accountingApiKey>` using:
    - `={{ 'Bearer ' + $('Workflow Configuration').first().json.accountingApiKey }}`
  - **Response format:** JSON.
- **Input/Output connections:** Input from **Workflow Configuration**; output to **Tax Calculation Agent**.
- **Version-specific requirements:** node typeVersion 4.3; behavior depends on n8n’s HTTP Request node v4+ (response settings structure).
- **Edge cases / failures:**
  - 401/403 if API key invalid.
  - 404/5xx if endpoint wrong/unavailable.
  - Non-JSON response will cause parsing issues (or empty JSON).
  - Data shape mismatch: the AI prompt expects fields like revenue/expenses/COGS; if the API returns different keys, the agent may produce wrong output or fail schema.

---

### Block 3 — Tax Calculation (AI Agent + Tools + Structured Output)
**Overview:** Uses GPT-5.2-Pro to compute tax metrics from financial inputs, enforcing arithmetic via Calculator tool and enforcing output via a structured JSON schema parser.  
**Nodes involved:**  
- Tax Calculation Agent  
- OpenAI Chat Model  
- Calculator Tool  
- Tax Report Output Parser  

#### Node: OpenAI Chat Model
- **Type / role:** `lmChatOpenAi` — LLM provider for the tax calculation agent.
- **Configuration choices:** Model set to **gpt-5.2-pro**.
- **Credentials:** `openAiApi` (must be configured).
- **Connections:** Provided to **Tax Calculation Agent** via `ai_languageModel`.
- **Edge cases / failures:**
  - Credential missing/expired.
  - Model not available in the account/region.
  - Rate limits/timeouts on long prompts or high load.

#### Node: Calculator Tool
- **Type / role:** `toolCalculator` — arithmetic tool the agent must use.
- **Configuration choices:** default (no parameters).
- **Connections:** Exposed to **Tax Calculation Agent** via `ai_tool`.
- **Edge cases:** Generally stable; failures are rare unless tool invocation is malformed.

#### Node: Tax Report Output Parser
- **Type / role:** `outputParserStructured` — forces a valid JSON object output.
- **Configuration choices:**
  - Manual JSON schema requiring:
    - `grossProfit` (number)
    - `netProfit` (number)
    - `taxableIncome` (number)
    - `taxLiability` (number)
    - `effectiveTaxRate` (number)
    - `calculationSteps` (string)
- **Connections:** Provided to **Tax Calculation Agent** via `ai_outputParser`.
- **Edge cases / failures:**
  - If the model outputs non-conforming JSON (missing fields, wrong types), parsing fails and the node/agent execution fails.

#### Node: Tax Calculation Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — orchestrates LLM + tools, returns structured tax calculation.
- **Configuration choices:**
  - **Input text to agent:**  
    `Financial data for tax calculation: <stringified $json>`
  - **System message:** instructs steps (gross profit, net profit, deductions, tax rate) and requires using Calculator tool for arithmetic; asks for step-by-step work and compliance with output parser format.
  - **Has output parser:** enabled (schema enforced via Tax Report Output Parser).
- **Connections:**  
  - Inputs: from **Fetch Financial Data** or from **Process Feedback and Corrections** (feedback loop).  
  - Tooling: **OpenAI Chat Model**, **Calculator Tool**, **Tax Report Output Parser**.  
  - Output: to **Format Pre-Filing Report** (main).
- **Edge cases / failures:**
  - Missing expected financial fields (e.g., `revenue`, `expenses`, `costOfGoodsSold`) yields incorrect results or schema errors.
  - Large financial JSON can exceed token limits.
  - If arithmetic isn’t actually delegated to the Calculator tool, outputs may be inconsistent—though the prompt requests it, enforcement is not absolute.

---

### Block 4 — Pre-Filing Report Formatting
**Overview:** Adds reporting metadata and a human-readable summary string for email/review.  
**Nodes involved:**  
- Format Pre-Filing Report  

#### Node: Format Pre-Filing Report
- **Type / role:** `set` — prepares report fields.
- **Configuration choices (fields set):**
  - `reportTitle`: `Corporate Tax Pre-Filing Report - <Month yyyy>` via `$now.format('MMMM yyyy')`
  - `companyName`: placeholder string (must be replaced)
  - `reportDate`: ISO timestamp via `$now.toISO()`
  - `reportSummary`: multiline string including:
    - Gross Profit, Net Profit, Taxable Income, Tax Liability
    - Effective Tax Rate shown as percent: `($json.effectiveTaxRate * 100) + '%'`
  - Include other fields: enabled (keeps calculated numbers and steps).
- **Connections:** Input from **Tax Calculation Agent**; output to **Submission Coordinator Agent**.
- **Edge cases / failures:**
  - If `effectiveTaxRate` is missing or null, expression may produce `NaN%` or fail depending on runtime coercion.
  - Currency formatting is simplistic (no rounding, locale, separators).

---

### Block 5 — Submission Coordination (AI Agent using Gmail + Drive tools)
**Overview:** A coordinator agent uses tools to email the report to a tax agent and archive the JSON report to Google Drive, then confirms completion.  
**Nodes involved:**  
- Submission Coordinator Agent  
- OpenAI Chat Model for Coordinator  
- Gmail Send Tool  
- Google Drive Archive Tool  

#### Node: OpenAI Chat Model for Coordinator
- **Type / role:** `lmChatOpenAi` — LLM provider for the coordinator agent.
- **Configuration:** Model **gpt-5.2-pro**.
- **Credentials:** same OpenAI credential.
- **Connections:** Provided to **Submission Coordinator Agent** via `ai_languageModel`.
- **Edge cases:** Same as other OpenAI model node.

#### Node: Gmail Send Tool
- **Type / role:** `gmailTool` — tool wrapper enabling the agent to send email.
- **Configuration choices:**
  - `sendTo`: `={{ $fromAI('recipient_email') }}`
  - `subject`: `={{ $fromAI('email_subject') }}`
  - `message`: `={{ $fromAI('email_body') }}`
- **Credentials:** Gmail OAuth2 credential required.
- **Connections:** Exposed as `ai_tool` to **Submission Coordinator Agent**.
- **Edge cases / failures:**
  - OAuth token expiry / missing scopes.
  - Gmail sending limits, blocked sending, invalid recipient address.
  - If the agent fails to supply required `fromAI` fields, tool invocation fails.

#### Node: Google Drive Archive Tool
- **Type / role:** `googleDriveTool` — tool wrapper enabling the agent to create/upload in Drive.
- **Configuration choices:**
  - `name`: `={{ $fromAI('file_name') }}`
  - `driveId`: “My Drive”
  - `folderId`: `={{ $fromAI('folder_id') }}`
- **Credentials:** Google Drive OAuth2 credential required.
- **Connections:** Exposed as `ai_tool` to **Submission Coordinator Agent**.
- **Important gap:** The tool node config does **not** specify file content/data mapping in this workflow JSON. As a result, the agent may “archive” only metadata or fail depending on how the tool expects content (this is a common integration pitfall with AI tools unless explicitly supported by the tool node).
- **Edge cases / failures:**
  - Folder ID invalid or not accessible to the credential user.
  - Missing `fromAI` fields (`file_name`, `folder_id`).
  - If content upload is required but not provided, archive may be empty or fail.

#### Node: Submission Coordinator Agent
- **Type / role:** LangChain Agent — orchestrates sending and archiving via tools.
- **Configuration choices:**
  - **Input text:** `Tax report data: <stringified $json>`
  - **System message:** instructs:
    1) email pre-filing report via Gmail tool (recipient should be `taxAgentEmail` from workflow data),
    2) archive JSON to Drive (folder should be `complianceVaultFolderId`),
    3) confirm success.
- **Connections:**
  - Input from **Format Pre-Filing Report**.
  - Tools: **Gmail Send Tool**, **Google Drive Archive Tool**.
  - LLM: **OpenAI Chat Model for Coordinator**.
  - Output: to **Check for Feedback**.
- **Edge cases / failures:**
  - The agent is instructed to use `taxAgentEmail` and `complianceVaultFolderId`, but the Gmail/Drive tools are wired to `$fromAI(...)`. This means the agent must correctly “decide” and populate those tool fields; there is no hard binding to the config variables.
  - If the tax agent email is still a placeholder, sending fails.
  - If the report JSON is large, email body may exceed practical limits.

---

### Block 6 — Feedback Handling, Recalculation Loop, and Finalization
**Overview:** Branches based on whether feedback has been received; if yes, merges corrections and recalculates tax; otherwise stamps final metadata.  
**Nodes involved:**  
- Check for Feedback  
- Process Feedback and Corrections  
- Finalize Submission Data  

#### Node: Check for Feedback
- **Type / role:** `if` — conditional routing.
- **Condition:** `={{ $json.feedbackReceived }}` is boolean true.
- **Connections (as wired):**
  - Outputs connect to **Process Feedback and Corrections** and **Finalize Submission Data**.
  - **Note:** The connections do not explicitly label true/false in the exported JSON; in n8n IF nodes, outputs are typically “true” and “false”. Verify in the canvas which branch goes where.
- **Edge cases / failures:**
  - If `feedbackReceived` is undefined/null/string, “loose” type validation may cause unexpected routing.
  - No upstream node sets `feedbackReceived` in this workflow; it must be injected by the coordinator agent output or the accounting API payload.

#### Node: Process Feedback and Corrections
- **Type / role:** `code` — merges corrections into the working dataset.
- **Logic (interpreted):**
  - For each input item:
    - Reads `feedback` (defaults to `'No specific feedback'`)
    - Reads `corrections` object (defaults to `{}`)
    - Produces `updatedData` merging:
      - original fields,
      - `correctionApplied: true`,
      - `feedbackNotes`,
      - and spreads `corrections` onto the root JSON (overriding existing fields).
- **Connections:** Input from IF; output loops back to **Tax Calculation Agent** for recalculation.
- **Edge cases / failures:**
  - If `corrections` contains unexpected keys, it can overwrite critical configuration or financial fields.
  - If corrections are nested but expected at root, recalculation agent may not see them.
  - No validation of correction values/types (could introduce NaN, strings instead of numbers, etc.).

#### Node: Finalize Submission Data
- **Type / role:** `set` — stamps final submission metadata.
- **Fields set:**
  - `submissionStatus`: `"Finalized"`
  - `finalizedDate`: current ISO timestamp
  - `submittedBy`: `"Automated Tax Filing System"`
  - `archiveConfirmation`: `"Report archived in compliance vault"`
  - Include other fields: enabled.
- **Connections:** Input from IF node; no downstream nodes (workflow ends here).
- **Edge cases / failures:**
  - This node claims “archived”, but archive success is not programmatically verified here; it relies on the coordinator agent’s tool execution and narrative confirmation.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Monthly Tax Filing Schedule | Schedule Trigger | Monthly workflow entry point | — | Workflow Configuration | ## How It Works\nThis workflow automates monthly tax filing processes by retrieving financial data, performing AI-driven tax calculations, coordinating pre-filing reviews with key stakeholders, incorporating feedback, and managing overall submission readiness. It pulls accounting records, executes GPT-5–based tax calculations with transparent reasoning, formats comprehensive pre-filing reports, and routes them to a submission coordinator via email for review. The system captures reviewer feedback through structured prompts, intelligently applies necessary corrections, archives finalized records in Google Drive, and continuously tracks filing status. It is designed for accounting firms, tax practices, and finance departments that require coordinated, multi-stakeholder tax filing with minimal manual intervention. |
| Workflow Configuration | Set | Stores API/recipient/tax constants | Monthly Tax Filing Schedule | Fetch Financial Data from Accounting Software | ## Fetch Financial Data\nWhat: Retrieves accounting records and financial statements from source systems.\nWhy: Ensures tax calculations are based on current.\n |
| Fetch Financial Data from Accounting Software | HTTP Request | Fetches monthly financial data from accounting API | Workflow Configuration | Tax Calculation Agent | ## Fetch Financial Data\nWhat: Retrieves accounting records and financial statements from source systems.\nWhy: Ensures tax calculations are based on current.\n |
| Tax Calculation Agent | LangChain Agent | Computes tax figures with tool-assisted arithmetic + structured output | Fetch Financial Data from Accounting Software; Process Feedback and Corrections | Format Pre-Filing Report | ## Calculate Tax Obligations\nWhat: Applies GPT-5 tax calculation with structured reasoning and detailed logic explanation.\nWhy: Delivers accurate, explainable tax figures with documented methodology.\n |
| OpenAI Chat Model | OpenAI Chat Model (LangChain) | LLM backend for Tax Calculation Agent | — | Tax Calculation Agent (ai_languageModel) | ## Calculate Tax Obligations\nWhat: Applies GPT-5 tax calculation with structured reasoning and detailed logic explanation.\nWhy: Delivers accurate, explainable tax figures with documented methodology.\n |
| Calculator Tool | LangChain Calculator Tool | Enforces arithmetic via tool calls | — | Tax Calculation Agent (ai_tool) | ## Calculate Tax Obligations\nWhat: Applies GPT-5 tax calculation with structured reasoning and detailed logic explanation.\nWhy: Delivers accurate, explainable tax figures with documented methodology.\n |
| Tax Report Output Parser | Structured Output Parser | Validates/forces JSON schema output | — | Tax Calculation Agent (ai_outputParser) | ## Calculate Tax Obligations\nWhat: Applies GPT-5 tax calculation with structured reasoning and detailed logic explanation.\nWhy: Delivers accurate, explainable tax figures with documented methodology.\n |
| Format Pre-Filing Report | Set | Builds title/date/company + summary text | Tax Calculation Agent | Submission Coordinator Agent | (blank) |
| Submission Coordinator Agent | LangChain Agent | Emails report + archives it via tools | Format Pre-Filing Report | Check for Feedback | (blank) |
| OpenAI Chat Model for Coordinator | OpenAI Chat Model (LangChain) | LLM backend for Submission Coordinator Agent | — | Submission Coordinator Agent (ai_languageModel) | (blank) |
| Gmail Send Tool | Gmail Tool | Sends email when invoked by agent | — | Submission Coordinator Agent (ai_tool) | (blank) |
| Google Drive Archive Tool | Google Drive Tool | Archives report when invoked by agent | — | Submission Coordinator Agent (ai_tool) | (blank) |
| Check for Feedback | IF | Routes based on `feedbackReceived` | Submission Coordinator Agent | Process Feedback and Corrections; Finalize Submission Data | ## Incorporate Feedback &  Stores Finalized Submission\nWhat: Captures coordinator feedback\nWhy: Ensures filing accuracy through collaborative review and maintains compliance |
| Process Feedback and Corrections | Code | Merges corrections into dataset for recalculation | Check for Feedback | Tax Calculation Agent | ## Incorporate Feedback &  Stores Finalized Submission\nWhat: Captures coordinator feedback\nWhy: Ensures filing accuracy through collaborative review and maintains compliance |
| Finalize Submission Data | Set | Stamps finalization metadata | Check for Feedback | — | ## Incorporate Feedback &  Stores Finalized Submission\nWhat: Captures coordinator feedback\nWhy: Ensures filing accuracy through collaborative review and maintains compliance |
| Sticky Note1 | Sticky Note | Canvas documentation | — | — | ## Prerequisites\nAccounting system access; OpenAI API key; Gmail account; Google Drive\n## Use Cases\nTax firms managing multi-client monthly filings with partner review  \n## Customization\nModify tax calculation prompts for jurisdictions, adjust feedback collection fields \n## Benefits\nEliminates manual filing coordination, reduces submission errors |
| Sticky Note2 | Sticky Note | Canvas documentation | — | — | ## Setup Steps\n1. Connect accounting system and configure financial data fetch parameters.\n2. Set up OpenAI GPT-5 API for tax calculations and reasoning extraction.\n3. Configure Gmail, Chat Model, and Google Drive credentials.\n4. Define submission coordinator contacts and configure feedback. |
| Sticky Note3 | Sticky Note | Canvas documentation | — | — | ## How It Works\nThis workflow automates monthly tax filing processes by retrieving financial data, performing AI-driven tax calculations, coordinating pre-filing reviews with key stakeholders, incorporating feedback, and managing overall submission readiness. It pulls accounting records, executes GPT-5–based tax calculations with transparent reasoning, formats comprehensive pre-filing reports, and routes them to a submission coordinator via email for review. The system captures reviewer feedback through structured prompts, intelligently applies necessary corrections, archives finalized records in Google Drive, and continuously tracks filing status. It is designed for accounting firms, tax practices, and finance departments that require coordinated, multi-stakeholder tax filing with minimal manual intervention. |
| Sticky Note4 | Sticky Note | Canvas documentation | — | — | ## Incorporate Feedback &  Stores Finalized Submission\nWhat: Captures coordinator feedback\nWhy: Ensures filing accuracy through collaborative review and maintains compliance |
| Sticky Note5 | Sticky Note | Canvas documentation | — | — | ## Calculate Tax Obligations\nWhat: Applies GPT-5 tax calculation with structured reasoning and detailed logic explanation.\nWhy: Delivers accurate, explainable tax figures with documented methodology.\n |
| Sticky Note6 | Sticky Note | Canvas documentation | — | — | ## Fetch Financial Data\nWhat: Retrieves accounting records and financial statements from source systems.\nWhy: Ensures tax calculations are based on current.\n |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name it: `GPT-5.2-Pro SME Corporate Tax Filing and Compliance Automation`
- (Optional) Set workflow to **Inactive** until credentials and placeholders are filled (the provided JSON is inactive).

2) **Add Trigger: “Monthly Tax Filing Schedule”**
- Node type: **Schedule Trigger**
- Set rule: **every 1 month**, at **09:00**
- This is the only entry point.

3) **Add Set node: “Workflow Configuration”**
- Node type: **Set**
- Add fields:
  - `accountingApiUrl` (String) → your accounting API endpoint (e.g., a “monthly summary” endpoint)
  - `accountingApiKey` (String) → API token/key
  - `taxAgentEmail` (String) → reviewer/tax agent email
  - `complianceVaultFolderId` (String) → Google Drive folder ID where archives should go
  - `taxRate` (Number) → `0.21` (or your jurisdiction)
  - `standardDeductions` (Number) → `5000` (or your policy)
- Enable **Include Other Fields**
- Connect: **Schedule Trigger → Workflow Configuration**

4) **Add HTTP Request node: “Fetch Financial Data from Accounting Software”**
- Node type: **HTTP Request**
- Method: GET (default) or match your API
- URL expression: use `accountingApiUrl` from the Set node:
  - `{{ $('Workflow Configuration').first().json.accountingApiUrl }}`
- Headers:
  - `Authorization` = `Bearer {{ $('Workflow Configuration').first().json.accountingApiKey }}`
- Response: **JSON**
- Connect: **Workflow Configuration → Fetch Financial Data...**

5) **Add AI components for tax calculation**
- Add node: **OpenAI Chat Model** (LangChain)
  - Model: `gpt-5.2-pro`
  - Credential: create/select **OpenAI API** credential in n8n.
- Add node: **Calculator Tool**
- Add node: **Structured Output Parser** named “Tax Report Output Parser”
  - Schema (manual) requiring:
    - `grossProfit` number
    - `netProfit` number
    - `taxableIncome` number
    - `taxLiability` number
    - `effectiveTaxRate` number
    - `calculationSteps` string

6) **Add LangChain Agent: “Tax Calculation Agent”**
- Node type: **AI Agent (LangChain)**
- Prompt type: “Define”
- Text input expression:
  - `Financial data for tax calculation: {{ JSON.stringify($json) }}`
- System message: use the workflow’s instructions (gross/net/taxable/tax liability; must use calculator; output per parser).
- Enable / attach:
  - Language model: **OpenAI Chat Model**
  - Tools: **Calculator Tool**
  - Output parser: **Tax Report Output Parser**
- Connect: **Fetch Financial Data... → Tax Calculation Agent**
- Connect the AI wiring:
  - OpenAI Chat Model → Tax Calculation Agent (as language model)
  - Calculator Tool → Tax Calculation Agent (as tool)
  - Output Parser → Tax Calculation Agent (as output parser)

7) **Add Set node: “Format Pre-Filing Report”**
- Node type: **Set**
- Enable **Include Other Fields**
- Add fields:
  - `reportTitle` = `Corporate Tax Pre-Filing Report - {{ $now.format('MMMM yyyy') }}`
  - `companyName` = your company name
  - `reportDate` = `{{ $now.toISO() }}`
  - `reportSummary` = build a multiline string using computed fields (grossProfit, netProfit, taxableIncome, taxLiability, effectiveTaxRate*100)
- Connect: **Tax Calculation Agent → Format Pre-Filing Report**

8) **Add AI components for submission coordination**
- Add node: **OpenAI Chat Model for Coordinator**
  - Model: `gpt-5.2-pro`
  - Credential: same OpenAI credential (or another)
- Add node: **Gmail Send Tool**
  - Credential: create/select **Gmail OAuth2** credential (must include send scope).
  - Map fields using AI variables:
    - To = `{{$fromAI('recipient_email')}}`
    - Subject = `{{$fromAI('email_subject')}}`
    - Body = `{{$fromAI('email_body')}}`
- Add node: **Google Drive Archive Tool**
  - Credential: create/select **Google Drive OAuth2** credential.
  - Drive: “My Drive”
  - File name = `{{$fromAI('file_name')}}`
  - Folder ID = `{{$fromAI('folder_id')}}`
  - (If you want actual JSON content archived reliably, add an explicit Drive “Upload” node or configure tool content if your n8n version supports it—otherwise the agent may not upload the report payload.)

9) **Add LangChain Agent: “Submission Coordinator Agent”**
- Text input: `Tax report data: {{ JSON.stringify($json) }}`
- System message: instruct email + archive using:
  - recipient from `taxAgentEmail`
  - folder from `complianceVaultFolderId`
  - file naming convention `Tax_Report_[Company]_[YYYY-MM].json`
  - content: full JSON report
  - confirm success
- Wire AI:
  - OpenAI Chat Model for Coordinator → Submission Coordinator Agent (language model)
  - Gmail Send Tool → Submission Coordinator Agent (tool)
  - Google Drive Archive Tool → Submission Coordinator Agent (tool)
- Connect: **Format Pre-Filing Report → Submission Coordinator Agent**

10) **Add IF node: “Check for Feedback”**
- Condition: Boolean “is true”
- Left value expression: `{{ $json.feedbackReceived }}`
- Connect: **Submission Coordinator Agent → Check for Feedback**

11) **Add Code node: “Process Feedback and Corrections”**
- Paste the provided JS logic to:
  - read `feedback` and `corrections`
  - merge corrections into the dataset
  - set `correctionApplied` and `feedbackNotes`
- Connect: **Check for Feedback (true branch) → Process Feedback and Corrections**
- Connect: **Process Feedback and Corrections → Tax Calculation Agent** (recalculation loop)

12) **Add Set node: “Finalize Submission Data”**
- Fields:
  - `submissionStatus` = `Finalized`
  - `finalizedDate` = `{{ $now.toISO() }}`
  - `submittedBy` = `Automated Tax Filing System`
  - `archiveConfirmation` = `Report archived in compliance vault`
- Enable **Include Other Fields**
- Connect: **Check for Feedback (false branch) → Finalize Submission Data**

13) **Credential checklist**
- OpenAI credential: valid API key with access to `gpt-5.2-pro`.
- Gmail OAuth2: account authorized, “send email” permissions/scopes.
- Google Drive OAuth2: permission to write into the compliance vault folder.

14) **Replace all placeholders**
- In Workflow Configuration and Format Pre-Filing Report:
  - Company name, accounting endpoint/key, tax agent email, Drive folder ID.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| **Prerequisites:** Accounting system access; OpenAI API key; Gmail account; Google Drive | From sticky note “Prerequisites” |
| **Use Cases:** Tax firms managing multi-client monthly filings with partner review | From sticky note “Use Cases” |
| **Customization:** Modify tax calculation prompts for jurisdictions, adjust feedback collection fields | From sticky note “Customization” |
| **Benefits:** Eliminates manual filing coordination, reduces submission errors | From sticky note “Benefits” |
| **Setup Steps:** 1) Connect accounting system and configure financial data fetch parameters. 2) Set up OpenAI GPT-5 API for tax calculations and reasoning extraction. 3) Configure Gmail, Chat Model, and Google Drive credentials. 4) Define submission coordinator contacts and configure feedback. | From sticky note “Setup Steps” |
| Feedback capture is implied but not implemented (no inbox watcher/webhook/form). You must add a mechanism to populate `feedbackReceived`, `feedback`, and `corrections`. | Observed workflow gap based on node graph |


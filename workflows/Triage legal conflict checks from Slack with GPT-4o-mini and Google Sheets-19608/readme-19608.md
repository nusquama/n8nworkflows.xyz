Triage legal conflict checks from Slack with GPT-4o-mini and Google Sheets

https://n8nworkflows.xyz/workflows/triage-legal-conflict-checks-from-slack-with-gpt-4o-mini-and-google-sheets-19608


# Triage legal conflict checks from Slack with GPT-4o-mini and Google Sheets

### 1. Workflow Overview

This workflow automates the legal conflict-check intake process by listening for requests in a Slack channel, verifying them against duplicate logs in Google Sheets, querying an OpenAI AI agent equipped with Model Context Protocol (MCP) search tools, posting a preliminary summary back into the original Slack thread, and recording an audit trail.

The workflow logic is categorized into the following functional blocks:

- **1.1 Input Reception & Normalization:** Captures incoming Slack messages, discards bot messages or empty text, and normalizes payload fields.
- **1.2 Duplicate Verification & Routing:** Generates a unique event key per message and queries a Google Sheets audit log to prevent processing duplicate requests.
- **1.3 AI Investigation & Tool Execution:** Feeds the unique request into an OpenAI agent configured with an MCP client tool connection, leveraging Google Sheets as a searchable knowledge base.
- **1.4 Classification & Response Formatting:** Parses the structured output from the AI agent to determine if a potential conflict exists, splitting the path to build either an alert message or a standard confirmation message.
- **1.5 Notification & Audit Logging:** Replies to the original Slack thread and logs execution states and request metadata back into Google Sheets.
- **1.6 MCP Server & Search Tools:** Exposes a separate Model Context Protocol (MCP) server cluster containing individual Google Sheets search tools (`Clients`, `Matters`, `Parties`, `RelatedEntities`) for the AI agent.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Input Reception & Normalization
**Overview:** This block triggers when a message is posted in the designated Slack channel, filtering out non-human or empty inputs and structuring data downstream.
**Nodes Involved:** 
- `When Slack Message Received`
- `Prepare Authenticated Input`

##### Node Details:
- **When Slack Message Received**
  - **Type & Technical Role:** `n8n-nodes-base.slackTrigger` — Event trigger listening for new Slack messages in a specific channel.
  - **Configuration:** Trigger set to `message` on channel ID `C0C0UEMPD5E` (`n8n-for-poc-purposes`). Uses webhook ID `slack-conflict-webhook`.
  - **Key Expressions/Variables:** None.
  - **Connections:** Output connects to `Prepare Authenticated Input`.
  - **Version Requirements:** Type version 1.
  - **Edge Cases/Failures:** Slack webhook delivery failures or revoked workspace OAuth scopes.

- **Prepare Authenticated Input**
  - **Type & Technical Role:** `n8n-nodes-base.code` — JavaScript code node used to filter and normalize event data.
  - **Configuration:** Drops execution if `bot_id`, `bot_message` subtype, or empty text is encountered. Returns standardized fields `chatInput`, `user`, `channel`, `ts`, and `originalEvent`.
  - **Key Expressions/Variables:** Uses `input.event ?? input` and `.trim()` on text.
  - **Connections:** Input from `When Slack Message Received`; output to `Generate Event Key`.
  - **Version Requirements:** Type version 2.
  - **Edge Cases/Failures:** Malformed payloads missing expected properties return empty arrays, gracefully halting the workflow.

---

#### Block 1.2: Duplicate Verification & Routing
**Overview:** Generates an immutable event identifier based on Slack metadata and cross-references an audit log sheet to check whether the request was already handled.
**Nodes Involved:**
- `Generate Event Key`
- `Read Events from Sheets`
- `Check Duplicate Status`

##### Node Details:
- **Generate Event Key**
  - **Type & Technical Role:** `n8n-nodes-base.code` — Code node running once per item to synthesize identifiers.
  - **Configuration:** Creates an `eventKey` string formatted as `slack:${channel}:${ts}` and flags `processingStatus` as `'new'`.
  - **Key Expressions/Variables:** `slack:${channel}:${ts}`.
  - **Connections:** Input from `Prepare Authenticated Input`; output to `Read Events from Sheets`.
  - **Version Requirements:** Type version 2.
  - **Edge Cases/Failures:** Throws an explicit error if channel or timestamp parameters are missing.

- **Read Events from Sheets**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheets` — Google Sheets lookup node.
  - **Configuration:** Queries spreadsheet ID `1DtF-sz7m2jXdvV4LO7uNoBaDIsnF5mnLV3m-PEG9Ks4` on sheet `ConflictAuditLog` (GID `20980903`), matching `event_key` against `eventKey`. Returns the first match with `alwaysOutputData` enabled.
  - **Key Expressions/Variables:** `={{ $json.eventKey }}`.
  - **Connections:** Input from `Generate Event Key`; output to `Check Duplicate Status`.
  - **Version Requirements:** Type version 4.7.
  - **Edge Cases/Failures:** Google Sheets API timeout or incorrect range mappings.

- **Check Duplicate Status**
  - **Type & Technical Role:** `n8n-nodes-base.if` — Conditional branching node.
  - **Configuration:** Evaluates whether the lookup returned an existing record (`Object.keys($json).length > 0`). 
  - **Key Expressions/Variables:** `={{ Object.keys($json).length > 0 }}`.
  - **Connections:** Input from `Read Events from Sheets`; True branch goes nowhere (halts), False branch outputs to `Prepare Agent Data Input`.
  - **Version Requirements:** Type version 2.3.
  - **Edge Cases/Failures:** False positives if sheet rows return empty metadata objects.

---

#### Block 1.3: AI Investigation & Tool Execution
**Overview:** Prepares the payload for the core AI agent, which uses an OpenAI chat model and an MCP client tool to query historical legal data.
**Nodes Involved:**
- `Prepare Agent Data Input`
- `Detect Conflict Agent`
- `OpenAI GPT-4 Model`
- `MCP Client Processor`

##### Node Details:
- **Prepare Agent Data Input**
  - **Type & Technical Role:** `n8n-nodes-base.code` — Code node to isolate necessary properties from the original payload.
  - **Configuration:** Extracts `chatInput`, `eventKey`, `user`, `channel`, `ts`, and `originalEvent`.
  - **Key Expressions/Variables:** References node data via `$('Generate Event Key')`.
  - **Connections:** Input from `Check Duplicate Status` (False branch); output to `Detect Conflict Agent`.
  - **Version Requirements:** Type version 2.
  - **Edge Cases/Failures:** Missing context references if upstream data is altered.

- **Detect Conflict Agent**
  - **Type & Technical Role:** `@n8n/n8n-nodes-langchain.agent` — Advanced AI agent node with tool invocation capabilities.
  - **Configuration:** Restricted system prompt defining strict tool-use limitations, prohibiting multi-tool calls or autonomous conflict clearings, and enforcing a structured text schema (`Status`, `Prospective client`, `Opposing party`, `Findings`, `Next step`). Maximum iterations set to 5.
  - **Key Expressions/Variables:** `={{ $json.chatInput }}`.
  - **Connections:** Inputs from `Prepare Agent Data Input`, `OpenAI GPT-4 Model` (`ai_languageModel`), and `MCP Client Processor` (`ai_tool`); output to `Analyze Conflict Data`.
  - **Version Requirements:** Type version 3.1.
  - **Edge Cases/Failures:** Agent looping or hitting iteration limits due to complex search queries.

- **OpenAI GPT-4 Model**
  - **Type & Technical Role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — Language model sub-node.
  - **Configuration:** Configured to use the `gpt-4o-mini` model.
  - **Key Expressions/Variables:** None.
  - **Connections:** Output connects to `Detect Conflict Agent` via `ai_languageModel`.
  - **Version Requirements:** Type version 1.3.
  - **Edge Cases/Failures:** OpenAI API rate limits, quota issues, or token length errors.

- **MCP Client Processor**
  - **Type & Technical Role:** `@n8n/n8n-nodes-langchain.mcpClientTool` — Model Context Protocol tool client sub-node.
  - **Configuration:** Connects to endpoint URL `https://n8n.intuz.net/mcp/conflict-tools`.
  - **Key Expressions/Variables:** None.
  - **Connections:** Output connects to `Detect Conflict Agent` via `ai_tool`.
  - **Version Requirements:** Type version 1.4.
  - **Edge Cases/Failures:** MCP server unavailability or network partition issues.

---

#### Block 1.4: Classification & Response Formatting
**Overview:** Parses the text output returned by the AI agent to extract entities, map status flags, and format corresponding Slack responses.
**Nodes Involved:**
- `Analyze Conflict Data`
- `Evaluate Potential Conflict`
- `Build Conflict Alert Message`
- `Build No Conflict Message`

##### Node Details:
- **Analyze Conflict Data**
  - **Type & Technical Role:** `n8n-nodes-base.code` — Code node for text parsing and extraction.
  - **Configuration:** Uses regular expressions to parse out client names, opposing parties, findings, next steps, and determine boolean conflict states based on output strings.
  - **Key Expressions/Variables:** `agentOutput.match(regex)` and string normalization methods.
  - **Connections:** Input from `Detect Conflict Agent`; output to `Evaluate Potential Conflict`.
  - **Version Requirements:** Type version 2.
  - **Edge Cases/Failures:** AI agent output deviating from expected text labels, causing fields to resolve as empty strings.

- **Evaluate Potential Conflict**
  - **Type & Technical Role:** `n8n-nodes-base.if` — Conditional routing node.
  - **Configuration:** Assesses whether `potentialConflict` equates to `false`.
  - **Key Expressions/Variables:** `={{ $json.potentialConflict }}` (against `false`).
  - **Connections:** Input from `Analyze Conflict Data`; True branch routes to `Build Conflict Alert Message`, False branch routes to `Build No Conflict Message`.
  - **Version Requirements:** Type version 2.3.
  - **Edge Cases/Failures:** Boolean type coercion issues if upstream parsing fails.

- **Build Conflict Alert Message**
  - **Type & Technical Role:** `n8n-nodes-base.code` — Code node assembling warning message structures.
  - **Configuration:** Prefixes output with alert iconography (`⚠️`) and mandatory human review disclosures.
  - **Key Expressions/Variables:** Pulls original channel and timestamp mappings.
  - **Connections:** Input from `Evaluate Potential Conflict` (True branch); output to `Post Slack Thread Reply`.
  - **Version Requirements:** Type version 2.
  - **Edge Cases/Failures:** Invalid Slack Block Kit or markup text.

- **Build No Conflict Message**
  - **Type & Technical Role:** `n8n-nodes-base.code` — Code node assembling clearance message structures.
  - **Configuration:** Prefixes output with confirmation iconography (`✅`) and preliminary search notices.
  - **Key Expressions/Variables:** References input items directly.
  - **Connections:** Input from `Evaluate Potential Conflict` (False branch); output to `Post Slack Thread Reply`.
  - **Version Requirements:** Type version 2.
  - **Edge Cases/Failures:** Unhandled string interpolations.

---

#### Block 1.5: Notification & Audit Logging
**Overview:** Delivers the finalized message back to the correct Slack discussion thread and appends an execution record to the Google Sheets audit tab.
**Nodes Involved:**
- `Post Slack Thread Reply`
- `Append Log to Sheets`

##### Node Details:
- **Post Slack Thread Reply**
  - **Type & Technical Role:** `n8n-nodes-base.slack` — Slack API node.
  - **Configuration:** Posts message content (`={{ $json.output }}`) to channel `={{ $('Prepare Authenticated Input').item.json.channel }}` replying inside thread timestamp `={{ $('Prepare Authenticated Input').item.json.ts }}`.
  - **Key Expressions/Variables:** Thread target mapping using upstream metadata.
  - **Connections:** Input from `Build Conflict Alert Message` and `Build No Conflict Message`; output to `Append Log to Sheets`.
  - **Version Requirements:** Type version 2.2.
  - **Edge Cases/Failures:** Slack token permission constraints or missing parent thread timestamps.

- **Append Log to Sheets**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheets` — Google Sheets append node.
  - **Configuration:** Appends rows to spreadsheet ID `1DtF-sz7m2jXdvV4LO7uNoBaDIsnF5mnLV3m-PEG9Ks4` on sheet `ConflictAuditLog` (GID `20980903`). Mappings populate `status`, `event_key`, `channel_id`, `created_at`, `message_ts`, `request_text`, `slack_user_id`, `potential_conflict`, and `prospective_client`.
  - **Key Expressions/Variables:** Uses `{{now}}` and expressions targeting execution data.
  - **Connections:** Input from `Post Slack Thread Reply`; terminal node.
  - **Version Requirements:** Type version 4.7.
  - **Edge Cases/Failures:** Sheet column header mismatches causing misplaced cell data.

---

#### Block 1.6: MCP Server & Search Tools
**Overview:** Exposes an internal Model Context Protocol (MCP) server cluster featuring discrete Google Sheets search tools to handle entity lookups.
**Nodes Involved:**
- `When MCP Server Activated`
- `Fetch Client Data`
- `Fetch Matter Data`
- `Fetch Party Data`
- `Fetch Related Entities Data`

##### Node Details:
- **When MCP Server Activated**
  - **Type & Technical Role:** `@n8n/n8n-nodes-langchain.mcpTrigger` — MCP server entrypoint trigger node.
  - **Configuration:** Path configured to `conflict-tools` using webhook ID `2b33cfa4-5c5e-40d4-9548-6f19f804a6fb`.
  - **Key Expressions/Variables:** None.
  - **Connections:** Receives incoming tool-calls from `Fetch Client Data`, `Fetch Matter Data`, `Fetch Party Data`, and `Fetch Related Entities Data` via `ai_tool`.
  - **Version Requirements:** Type version 2.1.
  - **Edge Cases/Failures:** Endpoint connectivity drops or malformed MCP client payloads.

- **Fetch Client Data**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheetsTool` — Google Sheets AI tool node.
  - **Configuration:** Searches sheet `Clients` (GID `0`) matching column `name` against `={{ $fromAI('query') }}`. Does not return only the first match.
  - **Key Expressions/Variables:** `={{ $fromAI('query') }}`.
  - **Connections:** Output connects to `When MCP Server Activated` via `ai_tool`.
  - **Version Requirements:** Type version 4.7.
  - **Edge Cases/Failures:** Empty query arguments passed by the AI agent.

- **Fetch Matter Data**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheetsTool` — Google Sheets AI tool node.
  - **Configuration:** Searches sheet `Matters` (GID `208591454`) matching column `name` against `={{ $fromAI('query') }}`.
  - **Key Expressions/Variables:** `={{ $fromAI('query') }}`.
  - **Connections:** Output connects to `When MCP Server Activated` via `ai_tool`.
  - **Version Requirements:** Type version 4.7.
  - **Edge Cases/Failures:** Sheet schema validation errors.

- **Fetch Party Data**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheetsTool` — Google Sheets AI tool node.
  - **Configuration:** Searches sheet `Parties` (GID `1841617652`) matching column `name` against `={{ $fromAI('query') }}` using an `OR` combination filter.
  - **Key Expressions/Variables:** `={{ $fromAI('query') }}`.
  - **Connections:** Output connects to `When MCP Server Activated` via `ai_tool`.
  - **Version Requirements:** Type version 4.7.
  - **Edge Cases/Failures:** Unmatched string formats during fuzzy lookups.

- **Fetch Related Entities Data**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheetsTool` — Google Sheets AI tool node.
  - **Configuration:** Searches sheet `RelatedEntities` (GID `856980659`) matching column `entity_name` against `={{ $fromAI('entity_name') }}`.
  - **Key Expressions/Variables:** `={{ $fromAI('entity_name') }}`.
  - **Connections:** Output connects to `When MCP Server Activated` via `ai_tool`.
  - **Version Requirements:** Type version 4.7.
  - **Edge Cases/Failures:** Missing entity name properties from LLM calls.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| When Slack Message Received | `n8n-nodes-base.slackTrigger` | Listens for new Slack messages | None | Prepare Authenticated Input | Receive Slack request |
| Prepare Authenticated Input | `n8n-nodes-base.code` | Validates and normalizes Slack event payload | When Slack Message Received | Generate Event Key | Receive Slack request |
| Generate Event Key | `n8n-nodes-base.code` | Builds unique event key and status flags | Prepare Authenticated Input | Read Events from Sheets | Check duplicate events |
| Read Events from Sheets | `n8n-nodes-base.googleSheets` | Queries sheet for existing event key | Generate Event Key | Check Duplicate Status | Check duplicate events |
| Check Duplicate Status | `n8n-nodes-base.if` | Evaluates whether event was already processed | Read Events from Sheets | Prepare Agent Data Input (False branch) | Check duplicate events |
| Prepare Agent Data Input | `n8n-nodes-base.code` | Prepares non-duplicate variables for AI agent | Check Duplicate Status | Detect Conflict Agent | Prepare AI investigation |
| Detect Conflict Agent | `@n8n/n8n-nodes-langchain.agent` | Runs LLM conflict intake analysis | Prepare Agent Data Input, OpenAI GPT-4 Model, MCP Client Processor | Analyze Conflict Data | Prepare AI investigation |
| OpenAI GPT-4 Model | `@n8n/n8n-nodes-langchain.lmChatOpenAi` | Language model provider for agent | None | Detect Conflict Agent | Prepare AI investigation |
| MCP Client Processor | `@n8n/n8n-nodes-langchain.mcpClientTool` | Connects agent to MCP server tools | None | Detect Conflict Agent | Prepare AI investigation |
| Analyze Conflict Data | `n8n-nodes-base.code` | Parses structured text output from agent | Detect Conflict Agent | Evaluate Potential Conflict | Classify and draft reply |
| Evaluate Potential Conflict | `n8n-nodes-base.if` | Routes workflow based on potential conflict status | Analyze Conflict Data | Build Conflict Alert Message, Build No Conflict Message | Classify and draft reply |
| Build Conflict Alert Message | `n8n-nodes-base.code` | Formats warning response for Slack | Evaluate Potential Conflict | Post Slack Thread Reply | Classify and draft reply |
| Build No Conflict Message | `n8n-nodes-base.code` | Formats clearance response for Slack | Evaluate Potential Conflict | Post Slack Thread Reply | Classify and draft reply |
| Post Slack Thread Reply | `n8n-nodes-base.slack` | Posts final analysis to original thread | Build Conflict Alert Message, Build No Conflict Message | Append Log to Sheets | Reply and log |
| Append Log to Sheets | `n8n-nodes-base.googleSheets` | Records audit details in Google Sheets | Post Slack Thread Reply | None | Reply and log |
| When MCP Server Activated | `@n8n/n8n-nodes-langchain.mcpTrigger` | Exposes MCP endpoint for conflict tools | Fetch Client Data, Fetch Matter Data, Fetch Party Data, Fetch Related Entities Data | None | Expose conflict tools |
| Fetch Client Data | `n8n-nodes-base.googleSheetsTool` | Tool: Search client records | None | When MCP Server Activated | Expose conflict tools |
| Fetch Matter Data | `n8n-nodes-base.googleSheetsTool` | Tool: Search matter records | None | When MCP Server Activated | Expose conflict tools |
| Fetch Party Data | `n8n-nodes-base.googleSheetsTool` | Tool: Search party records | None | When MCP Server Activated | Expose conflict tools |
| Fetch Related Entities Data | `n8n-nodes-base.googleSheetsTool` | Tool: Search related entities | None | When MCP Server Activated | Expose conflict tools |

---

### 4. Reproducing the Workflow from Scratch

#### Step 1: Create the Main Flow Nodes
1. **When Slack Message Received (`n8n-nodes-base.slackTrigger`)**: Set trigger type to `message`. Bind Slack credentials and select channel `C0C0UEMPD5E`. Set webhook ID to `slack-conflict-webhook`.
2. **Prepare Authenticated Input (`n8n-nodes-base.code`)**: Add a JavaScript node to drop bot/empty messages and return an object with `chatInput`, `user`, `channel`, `ts`, and `originalEvent`.
3. **Generate Event Key (`n8n-nodes-base.code`)**: Add a code node to create an `eventKey` property using `slack:${channel}:${ts}` and set `processingStatus` to `'new'`.
4. **Read Events from Sheets (`n8n-nodes-base.googleSheets`)**: Connect Google Sheets credentials. Specify Document ID `1DtF-sz7m2jXdvV4LO7uNoBaDIsnF5mnLV3m-PEG9Ks4` and Sheet name `ConflictAuditLog`. Set operation to `Lookup`, mapping filter column `event_key` to `={{ $json.eventKey }}`. Enable `Always Output Data`.
5. **Check Duplicate Status (`n8n-nodes-base.if`)**: Set condition to verify that `Object.keys($json).length > 0` evaluates to `true`. Route the `false` path to the next step.

#### Step 2: Configure the AI Investigation Agent
6. **Prepare Agent Data Input (`n8n-nodes-base.code`)**: Pass through `chatInput`, `eventKey`, `user`, `channel`, `ts`, and `originalEvent` from upstream node references.
7. **OpenAI GPT-4 Model (`@n8n/n8n-nodes-langchain.lmChatOpenAi`)**: Connect OpenAI credentials, choose model `gpt-4o-mini`, and link to the agent via `ai_languageModel`.
8. **MCP Client Processor (`@n8n/n8n-nodes-langchain.mcpClientTool`)**: Set endpoint URL to `https://n8n.intuz.net/mcp/conflict-tools` and link to the agent via `ai_tool`.
9. **Detect Conflict Agent (`@n8n/n8n-nodes-langchain.agent`)**: Set prompt type to `define`, pass input `={{ $json.chatInput }}`, configure max iterations to 5, and insert the strict legal intake prompt constraining tool usage and output formatting.

#### Step 3: Parse and Route Classification
10. **Analyze Conflict Data (`n8n-nodes-base.code`)**: Add a JavaScript node using regular expressions to extract `Prospective client`, `Opposing party`, `Findings`, and `Next step`, and evaluate whether the text contains `"status: potential conflict identified"`.
11. **Evaluate Potential Conflict (`n8n-nodes-base.if`)**: Check if `potentialConflict` is `false`.
12. **Build Conflict Alert Message (`n8n-nodes-base.code`)**: Format output text with warning headers (`⚠️`) and mandatory review notices for the `true` branch of the IF node.
13. **Build No Conflict Message (`n8n-nodes-base.code`)**: Format output text with confirmation headers (`✅`) and preliminary search disclaimers for the `false` branch of the IF node.

#### Step 4: Deliver Responses and Log Audit Data
14. **Post Slack Thread Reply (`n8n-nodes-base.slack`)**: Select action type, target channel from upstream input, set message text to `={{ $json.output }}`, and configure reply thread timestamp (`thread_ts`) using original event metadata.
15. **Append Log to Sheets (`n8n-nodes-base.googleSheets`)**: Set operation to `Append`, use the same spreadsheet ID and `ConflictAuditLog` sheet, and map schema fields (`status`, `event_key`, `channel_id`, `created_at`, `message_ts`, `request_text`, `slack_user_id`, `potential_conflict`, `prospective_client`) to their corresponding data expressions.

#### Step 5: Setup the Separate MCP Server Workflow
16. **When MCP Server Activated (`@n8n/n8n-nodes-langchain.mcpTrigger`)**: Create a separate workflow path with path parameter `conflict-tools` and webhook ID `2b33cfa4-5c5e-40d4-9548-6f19f804a6fb`.
17. **Google Sheets Tool Nodes (`n8n-nodes-base.googleSheetsTool`)**: Create four tool nodes (`Fetch Client Data`, `Fetch Matter Data`, `Fetch Party Data`, `Fetch Related Entities Data`) mapped to sheets `Clients` (GID 0), `Matters` (GID 208591454), `Parties` (GID 1841617652), and `RelatedEntities` (GID 856980659). Connect their outputs to the MCP trigger via `ai_tool` channels, assigning unique query mapping expressions (`$fromAI('query')` or `$fromAI('entity_name')`).

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Website Template Reference | https://www.intuz.com/n8n-workflow-automation-templates/ |
| Support Contact Email | getstarted@intuz.com |
| Company LinkedIn Page | https://www.linkedin.com/company/intuz |
| Partner Links & Setup | https://n8n.partnerlinks.io/intuz |
| Custom Workflow Automation Requests | https://www.intuz.com/get-started/ |
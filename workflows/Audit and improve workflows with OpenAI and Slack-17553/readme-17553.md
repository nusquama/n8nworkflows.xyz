Audit and improve workflows with OpenAI and Slack

https://n8nworkflows.xyz/workflows/audit-and-improve-workflows-with-openai-and-slack-17553


# Audit and improve workflows with OpenAI and Slack

### 1. Workflow Overview

This workflow automates the weekly auditing of all n8n workflows within an instance using OpenAI. It inspects workflow layouts, dynamically documents them by generating and collision-resolving sticky notes, evaluates code quality, security, reliability, and maintainability, posts rich report cards to Slack, and tracks user preferences via Slack reaction toggles stored in an n8n Data Table.

The logic is categorized into the following functional blocks:
- **1.1 Initialization & Setup:** Handles manual initialization of the tracking Data Table and weekly schedule triggering to fetch all instance workflows.
- **1.2 Data Filtering & Merging:** Compares retrieved workflows against excluded items in the n8n Data Table to determine which items require scanning.
- **1.3 AI Layout Documentation & Collision Resolution:** Strips existing documentation, uses an OpenAI agent with a structured output parser to logically group canvas nodes, calculates dynamic bounding boxes, resolves layout collisions, and embeds structured sticky notes into the workflow representation.
- **1.4 AI Quality Analysis & Slack Reporting:** Parses the refined workflow representation, sends structural metrics and code context to OpenAI for scoring and issue detection, posts interactive block messages to Slack, and upserts tracking records to the Data Table.
- **1.5 Slack Reaction Event Handler:** Listens for incoming Slack emoji reactions (`:x:`) to dynamically enable or disable future scans for specific workflows and posts confirmation feedback to the channel.

---

### 2. Block-by-Block Analysis

---

### Block 1.1: Initialization & Setup
**Overview:**  
Provides entry points to manually bootstrap the required tracking Data Table or trigger automated weekly execution sweeps across the n8n instance.

**Nodes Involved:**
- `When clicking ‘Execute workflow’`
- `Create a data table`
- `Schedule Trigger`

**Node Details:**

- **When clicking ‘Execute workflow’**
  - **Type & Technical Role:** `n8n-nodes-base.manualTrigger` — Manual execution entry point.
  - **Configuration:** Default parameters (no configuration required).
  - **Key Expressions:** None.
  - **Connections:** Input: None | Output: `Create a data table`.
  - **Edge Cases:** Run only once during initial setup to initialize the storage infrastructure.

- **Create a data table**
  - **Type & Technical Role:** `n8n-nodes-base.dataTable` — Infrastructure setup node.
  - **Configuration:** Creates an n8n Data Table named `AI Workflow Improvements` containing columns: `workflowName`, `workflowId`, `scan` (boolean), and `lastSlackMessageTimestampId`.
  - **Key Expressions:** None.
  - **Connections:** Input: `When clicking ‘Execute workflow’` | Output: None.
  - **Edge Cases:** Fails if a table with the exact name already exists or if administrative API privileges are missing.

- **Schedule Trigger**
  - **Type & Technical Role:** `n8n-nodes-base.scheduleTrigger` — Cron-like automated scheduler.
  - **Configuration:** Configured to trigger on an interval of every 7 days at 16:00.
  - **Key Expressions:** None.
  - **Connections:** Input: None | Output: `Get many workflows`.
  - **Edge Cases:** Requires the n8n instance to be running continuously.

---

### Block 1.2: Data Filtering & Merging
**Overview:**  
Retrieves all workflows registered on the n8n instance and cross-references them against excluded workflows stored in the Data Table.

**Nodes Involved:**
- `Get many workflows`
- `Get known workflows`
- `Merge`
- `Loop Over Items`

**Node Details:**

- **Get many workflows**
  - **Type & Technical Role:** `n8n-nodes-base.n8n` — n8n API integration node.
  - **Configuration:** Fetches workflow definitions using the instance API.
  - **Credentials:** Uses `N8N API Key` (`WlI5sSDhwn6o3goZ`).
  - **Key Expressions:** None.
  - **Connections:** Input: `Schedule Trigger` | Output: `Get known workflows`, `Merge` (Input 1).
  - **Edge Cases:** Invalid API key or insufficient instance permissions will return authorization errors.

- **Get known workflows**
  - **Type & Technical Role:** `n8n-nodes-base.dataTable` — Data Table query node.
  - **Configuration:** Retrieves records from `AI Workflow Improvements` where `scan` is `false`. Executes once per run (`executeOnce: true`).
  - **Key Expressions:** None.
  - **Connections:** Input: `Get many workflows` | Output: `Merge` (Input 2).
  - **Edge Cases:** Table ID mismatch causes query failure.

- **Merge**
  - **Type & Technical Role:** `n8n-nodes-base.merge` — Data combination node.
  - **Configuration:** Advanced join mode (`keepNonMatches`) merging inputs on workflow ID fields (`id` vs `workflowId`), outputting items from Input 1 that do not match excluded filters.
  - **Key Expressions:** None.
  - **Connections:** Input 1: `Get many workflows` | Input 2: `Get known workflows` | Output: `Loop Over Items`.
  - **Edge Cases:** Unmatched schema fields can lead to dropped items.

- **Loop Over Items**
  - **Type & Technical Role:** `n8n-nodes-base.splitInBatches` — Batch iterator node.
  - **Configuration:** Processes workflow items iteratively.
  - **Key Expressions:** None.
  - **Connections:** Input: `Merge` | Outputs: Branch 1 (`Filter` / execution check), Branch 2 (`Strip & Prepare`).
  - **Edge Cases:** Large workflow arrays can result in prolonged execution times.

---

### Block 1.3: AI Layout Documentation & Collision Resolution
**Overview:**  
Prepares the workflow canvas structure, executes an OpenAI agent to cluster nodes logically, computes bounding boxes, resolves visual overlaps, and injects updated sticky notes.

**Nodes Involved:**
- `Strip & Prepare`
- `Parse Nodes`
- `AI Groups Logically`
- `OpenAI Chat Model`
- `Structured Output Parser`
- `Compute Bounding Boxes`
- `Collision Resolution`
- `Generate Stickies`
- `Merge & Export`
- `Collision Detector`

**Node Details:**

- **Strip & Prepare**
  - **Type & Technical Role:** `n8n-nodes-base.code` — JavaScript execution node.
  - **Configuration:** Removes existing sticky notes, maps AI sub-nodes (`ai_languageModel`, `ai_tool`, etc.), and calculates input/output slot indices.
  - **Key Expressions:** Processes incoming workflow JSON payload.
  - **Connections:** Input: `Loop Over Items` | Output: `Parse Nodes`.
  - **Edge Cases:** Malformed connections definitions can throw runtime parsing errors.

- **Parse Nodes**
  - **Type & Technical Role:** `n8n-nodes-base.code` — JavaScript execution node.
  - **Configuration:** Computes node dimensions, extracts context strings based on simple node types, and compiles enriched node metadata.
  - **Key Expressions:** None.
  - **Connections:** Input: `Strip & Prepare` | Output: `AI Groups Logically`.
  - **Edge Cases:** Unrecognized node types fall back to generic descriptions.

- **AI Groups Logically**
  - **Type & Technical Role:** `@n8n/n8n-nodes-langchain.agent` — LangChain AI Agent node.
  - **Configuration:** Uses a system prompt instructing the model to group nodes based on logical purpose and spatial canvas coordinates.
  - **Credentials:** Uses `OpenAi Api Key` (`U8Wgd5xDxyUFsl1V`) via linked chat model.
  - **Key Expressions:** `=Workflow: {{ $json.workflowName }}`, `Node count: {{ $json.nodeCount }}`.
  - **Connections:** Input: `Parse Nodes`, `OpenAI Chat Model` | Output: `Compute Bounding Boxes`.
  - **Edge Cases:** Model response truncation or schema violations handled by the structured output parser.

- **OpenAI Chat Model**
  - **Type & Technical Role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — Language model sub-node.
  - **Configuration:** Model set to `gpt-5.4-nano`.
  - **Credentials:** Uses `OpenAi Api Key` (`U8Wgd5xDxyUFsl1V`).
  - **Connections:** Connected via `ai_languageModel` to `AI Groups Logically` and `Structured Output Parser`.

- **Structured Output Parser**
  - **Type & Technical Role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — LangChain parser sub-node.
  - **Configuration:** Enforces a strict JSON schema containing `mainOverview` and `groups` arrays.
  - **Connections:** Connected via `ai_outputParser` to `AI Groups Logically`.

- **Compute Bounding Boxes**
  - **Type & Technical Role:** `n8n-nodes-base.code` — JavaScript execution node.
  - **Configuration:** Calculates geometric bounding boxes around grouped nodes and estimates dynamic sticky note text heights.
  - **Key Expressions:** Reads parser output and node dimensions.
  - **Connections:** Input: `AI Groups Logically` | Output: `Collision Resolution`.
  - **Edge Cases:** Zero-node groups are filtered out to prevent division errors.

- **Collision Resolution**
  - **Type & Technical Role:** `n8n-nodes-base.code` — JavaScript execution node.
  - **Configuration:** Iteratively checks sticky bounding box overlaps and pushes overlapping groups apart along the axis of least displacement.
  - **Key Expressions:** None.
  - **Connections:** Input: `Compute Bounding Boxes` | Output: `Generate Stickies`.
  - **Edge Cases:** Max iteration guard (`MAX_ITERATIONS = 15`) prevents infinite loops.

- **Generate Stickies**
  - **Type & Technical Role:** `n8n-nodes-base.code` — JavaScript execution node.
  - **Configuration:** Generates UUIDs and formatting structures for the main overview sticky note and colored group sticky notes.
  - **Key Expressions:** None.
  - **Connections:** Input: `Collision Resolution` | Output: `Merge & Export`.
  - **Edge Cases:** Text estimation length discrepancies can occasionally cause tight line wraps.

- **Merge & Export**
  - **Type & Technical Role:** `n8n-nodes-base.code` — JavaScript execution node.
  - **Configuration:** Replaces old sticky notes with newly generated ones, updates shifted node coordinates, and compiles the updated workflow JSON.
  - **Key Expressions:** References `Loop Over Items` for original workflow payload.
  - **Connections:** Input: `Generate Stickies` | Output: `Collision Detector`.
  - **Edge Cases:** Missing node coordinates default to origin positions.

- **Collision Detector**
  - **Type & Technical Role:** `n8n-nodes-base.code` — JavaScript execution node.
  - **Configuration:** Performs a secondary safety check validating that no foreign nodes overlap unrelated sticky notes.
  - **Key Expressions:** None.
  - **Connections:** Input: `Merge & Export` | Output: `Parse for analysis`.
  - **Edge Cases:** Complex dense layouts may require multiple adjustment passes.

---

### Block 1.4: AI Quality Analysis & Slack Reporting
**Overview:**  
Performs code and architecture analysis using OpenAI, posts formatted reports to Slack, waits briefly, and updates tracking records in the Data Table.

**Nodes Involved:**
- `Parse for analysis`
- `AI analysis`
- `OpenAI Chat Model2`
- `Structured Output Parser 2`
- `Filter`
- `Send a message`
- `Add a reaction`
- `Wait`
- `Upsert row(s)`

**Node Details:**

- **Parse for analysis**
  - **Type & Technical Role:** `n8n-nodes-base.code` — JavaScript execution node.
  - **Configuration:** Extracts simplified types, parameter context, and expression references from the updated workflow JSON.
  - **Key Expressions:** None.
  - **Connections:** Input: `Collision Detector` | Output: `AI analysis`.
  - **Edge Cases:** Deeply nested parameter structures can return empty fallback descriptions.

- **AI analysis**
  - **Type & Technical Role:** `@n8n/n8n-nodes-langchain.agent` — LangChain AI Agent node.
  - **Configuration:** Evaluates reliability, security, maintainability, and design metrics based on comprehensive expert auditing rules.
  - **Credentials:** Uses `OpenAi Api Key` (`U8Wgd5xDxyUFsl1V`).
  - **Key Expressions:** `=Workflow name: {{ $json.workflowName }}`, `Node count: {{ $json.nodeCount }}`.
  - **Connections:** Input: `Parse for analysis`, `OpenAI Chat Model2` | Output: `Loop Over Items` (loop continuation).
  - **Edge Cases:** Large workflow definitions can hit token limits.

- **OpenAI Chat Model2**
  - **Type & Technical Role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — Language model sub-node.
  - **Configuration:** Model set to `gpt-4o-mini`.
  - **Credentials:** Uses `OpenAi Api Key` (`U8Wgd5xDxyUFsl1V`).
  - **Connections:** Connected via `ai_languageModel` to `AI analysis` and `Structured Output Parser 2`.

- **Structured Output Parser 2**
  - **Type & Technical Role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — LangChain parser sub-node.
  - **Configuration:** Enforces output schema for scores (`reliability`, `security`, `maintainability`, `design`), summaries, workflow-level issues, and node-level issues.
  - **Connections:** Connected via `ai_outputParser` to `AI analysis`.

- **Filter**
  - **Type & Technical Role:** `n8n-nodes-base.filter` — Data filter node.
  - **Configuration:** Validates that incoming evaluation payloads are not empty.
  - **Key Expressions:** `={{ $json }}` (not empty condition).
  - **Connections:** Input: `Loop Over Items` | Output: `Send a message`.
  - **Edge Cases:** Drops execution items if the analysis object is null.

- **Send a message**
  - **Type & Technical Role:** `n8n-nodes-base.slack` — Slack integration action node.
  - **Configuration:** Posts rich Block Kit messages containing headers, summary text, scoring tables, workflow issues, and node-level issue carousels to channel `workflow-improvements` (`C0BCJ6ZL23T`).
  - **Credentials:** Uses `Slack Api Key` (`IvZCxNWZUk2WLNfY`).
  - **KeyExpressions:** Extensive Block Kit JSON templates parsing `{{$json.output.summary}}`, scores, and issues.
  - **Connections:** Input: `Filter` | Output: `Add a reaction`.
  - **Edge Cases:** Slack API rate limits or invalid channel IDs will cause delivery failure.

- **Add a reaction**
  - **Type & Technical Role:** `n8n-nodes-base.slack` — Slack integration action node.
  - **Configuration:** Adds an `:x:` reaction to the posted audit report message.
  - **Credentials:** Uses `Slack OAuth2` (`xUHEPQEMBq3flz7M`).
  - **Key Expressions:** `={{ $json.channel }}`, `={{ $json.message_timestamp }}`.
  - **Connections:** Input: `Send a message` | Output: `Wait`.
  - **Edge Cases:** Requires `reactions:write` scope on Slack OAuth credentials.

- **Wait**
  - **Type & Technical Role:** `n8n-nodes-base.wait` — Execution pause node.
  - **Configuration:** Pauses execution briefly to prevent race conditions with webhook triggers.
  - **Key Expressions:** None.
  - **Connections:** Input: `Add a reaction` | Output: `Upsert row(s)`.
  - **Edge Cases:** Insufficient wait duration can trigger downstream Slack event loops.

- **Upsert row(s)**
  - **Type & Technical Role:** `n8n-nodes-base.dataTable` — Data Table upsert node.
  - **Configuration:** Upserts workflow tracking data (`workflowId`, `workflowName`, `scan: true`, `lastSlackMessageTimestampId`) into `AI Workflow Improvements`.
  - **Key Expressions:** `={{ $('Loop Over Items').item.json.output.workflowId }}`, `={{ $('Send a message').item.json.message_timestamp }}`.
  - **Connections:** Input: `Wait` | Output: None.
  - **Edge Cases:** Matching column configuration mismatch causes duplicate rows.

---

### Block 1.5: Slack Reaction Event Handler
**Overview:**  
Listens for Slack reaction events, determines if a user added or removed an `:x:` reaction on a report message, and updates the Data Table to disable or enable scanning accordingly.

**Nodes Involved:**
- `Slack Trigger`
- `If reaction is :x:`
- `Switch`
- `Disable scan`
- `Enable scan`
- `Notify about disabling scan`
- `Notify about enabling scan`

**Node Details:**

- **Slack Trigger**
  - **Type & Technical Role:** `n8n-nodes-base.slackTrigger` — Slack webhook trigger node.
  - **Configuration:** Listens to `any_event` on channel `workflow-improvements` (`C0BCJ6ZL23T`).
  - **Credentials:** Uses `Slack OAuth2` (`xUHEPQEMBq3flz7M`).
  - **Key Expressions:** None.
  - **Connections:** Input: None | Output: `If reaction is :x:`.
  - **Edge Cases:** Webhook subscriptions must be properly configured in the Slack App settings.

- **If reaction is :x:**
  - **Type & Technical Role:** `n8n-nodes-base.if` — Conditional branch node.
  - **Configuration:** Checks if the event reaction equals `x`.
  - **Key Expressions:** `={{ $json.reaction }}` equals `x`.
  - **Connections:** Input: `Slack Trigger` | Output: `Switch`.
  - **Edge Cases:** Ignores non-matching emojis.

- **Switch**
  - **Type & Technical Role:** `n8n-nodes-base.switch` — Multi-branch routing node.
  - **Configuration:** Routes based on event type (`reaction_added` vs `reaction_removed`).
  - **Key Expressions:** `={{ $json.type }}`.
  - **Connections:** Input: `If reaction is :x:` | Output 1: `Disable scan`, Output 2: `Enable scan`.
  - **Edge Cases:** Unhandled reaction subtypes pass through silently.

- **Disable scan**
  - **Type & Technical Role:** `n8n-nodes-base.dataTable` — Data Table update node.
  - **Configuration:** Updates rows where `lastSlackMessageTimestampId` matches message timestamp, setting `scan` to `false`.
  - **Key Expressions:** `={{ $json.item.ts }}`.
  - **Connections:** Input: `Switch` (Branch 1) | Output: `Notify about disabling scan`.
  - **Edge Cases:** Timestamp mismatch fails to update database state.

- **Enable scan**
  - **Type & Technical Role:** `n8n-nodes-base.dataTable` — Data Table update node.
  - **Configuration:** Updates rows where `lastSlackMessageTimestampId` matches message timestamp, setting `scan` to `true`.
  - **Key Expressions:** `={{ $json.item.ts }}`.
  - **Connections:** Input: `Switch` (Branch 2) | Output: `Notify about enabling scan`.
  - **Edge Cases:** Timestamp mismatch fails to update database state.

- **Notify about disabling scan**
  - **Type & Technical Role:** `n8n-nodes-base.slack` — Slack integration action node.
  - **Configuration:** Posts confirmation message to Slack when scanning is disabled.
  - **Credentials:** Uses `Slack OAuth2` (`xUHEPQEMBq3flz7M`).
  - **Key Expressions:** `=<@{{ $('Slack Trigger').item.json.user }}> disabled scan for {{ $json.workflowName }} :x:`
  - **Connections:** Input: `Disable scan` | Output: None.
  - **Edge Cases:** Requires proper bot token scopes.

- **Notify about enabling scan**
  - **Type & Technical Role:** `n8n-nodes-base.slack` — Slack integration action node.
  - **Configuration:** Posts confirmation message to Slack when scanning is re-enabled.
  - **Credentials:** Uses `Slack OAuth2` (`xUHEPQEMBq3flz7M`).
  - **Key Expressions:** `=<@{{ $('Slack Trigger').item.json.user }}> enabled scan for {{ $json.workflowName }} :white_check_mark:`
  - **Connections:** Input: `Enable scan` | Output: None.
  - **Edge Cases:** Requires proper bot token scopes.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Sticky Note8` | `n8n-nodes-base.stickyNote` | Documentation note | None | None | ## Propose Workflows Improvement with AI<br><br>### How it works<br>This workflow checks all your workflows for possible improvements, and presents you with suggested changes. It doesn't edit your workflow, it scans it and tries to find places for improvement<br><br>### Setup steps<br>- [ ] Run workflow with 'Execute workflow' to create Data Table<br>- [ ] Ensure that Slack nodes are configured with correct credentials and channels.<br>- [ ] Set up OpenAI credentials for nodes where AI processing is required.<br><br>### Customization<br>You can customize the node processing logic to suit specific workflows or modify AI behavior for different output requirements.<br>You can also play with Slack blocks used for message being sent to the channel or change emote that is used for reaction<br><br>Need help? Contact us [here](https://sailingbyte.com/contact/) or visit [sailingbyte.com](https://sailingbyte.com)!<br><br>Happy hacking! |
| `Sticky Note10` | `n8n-nodes-base.stickyNote` | Documentation note | None | None | ## Run this first!<br><br>Running this part of the workflow will create data table used by the workflow |
| `When clicking ‘Execute workflow’` | `n8n-nodes-base.manualTrigger` | Manual trigger | None | `Create a data table` | ## Run this first!<br><br>Running this part of the workflow will create data table used by the workflow |
| `Create a data table` | `n8n-nodes-base.dataTable` | Create table | `When clicking ‘Execute workflow’` | None | ## Run this first!<br><br>Running this part of the workflow will create data table used by the workflow |
| `Schedule Trigger` | `n8n-nodes-base.scheduleTrigger` | Schedule trigger | None | `Get many workflows` | ## Fetch the workflows<br>Fetch workflows, filter out those that shouldn't be scanned, and loop over them |
| `Get many workflows` | `n8n-nodes-base.n8n` | Fetch workflows | `Schedule Trigger` | `Get known workflows`, `Merge` | ## Fetch the workflows<br>Fetch workflows, filter out those that shouldn't be scanned, and loop over them |
| `Get known workflows` | `n8n-nodes-base.dataTable` | Query table | `Get many workflows` | `Merge` | ## Fetch the workflows<br>Fetch workflows, filter out those that shouldn't be scanned, and loop over them |
| `Merge` | `n8n-nodes-base.merge` | Merge data | `Get many workflows`, `Get known workflows` | `Loop Over Items` | ## Fetch the workflows<br>Fetch workflows, filter out those that shouldn't be scanned, and loop over them |
| `Loop Over Items` | `n8n-nodes-base.splitInBatches` | Loop iterator | `Merge` | `Filter`, `Strip & Prepare` | ## Fetch the workflows<br>Fetch workflows, filter out those that shouldn't be scanned, and loop over them |
| `Strip & Prepare` | `n8n-nodes-base.code` | Prepare nodes | `Loop Over Items` | `Parse Nodes` | ## Prepare and parse nodes<br><br>Prepares nodes and parses them for further processing. |
| `Parse Nodes` | `n8n-nodes-base.code` | Parse nodes | `Strip & Prepare` | `AI Groups Logically` | ## Prepare and parse nodes<br><br>Prepares nodes and parses them for further processing. |
| `AI Groups Logically` | `@n8n/n8n-nodes-langchain.agent` | AI clustering | `Parse Nodes`, `OpenAI Chat Model`, `Structured Output Parser` | `Compute Bounding Boxes` | ## AI logical grouping<br><br>Uses AI to logically group nodes. |
| `OpenAI Chat Model` | `@n8n/n8n-nodes-langchain.lmChatOpenAi` | LLM service | None | `AI Groups Logically`, `Structured Output Parser` | ## AI logical grouping<br><br>Uses AI to logically group nodes. |
| `Structured Output Parser` | `@n8n/n8n-nodes-langchain.outputParserStructured` | Output parser | `OpenAI Chat Model` | `AI Groups Logically` | ## AI logical grouping<br><br>Uses AI to logically group nodes. |
| `Compute Bounding Boxes` | `n8n-nodes-base.code` | Compute bounds | `AI Groups Logically` | `Collision Resolution` | ## Collision handling and export<br><br>Handles collisions and merges results for export. |
| `Collision Resolution` | `n8n-nodes-base.code` | Resolve collisions | `Compute Bounding Boxes` | `Generate Stickies` | ## Collision handling and export<br><br>Handles collisions and merges results for export. |
| `Generate Stickies` | `n8n-nodes-base.code` | Generate stickies | `Collision Resolution` | `Merge & Export` | ## Collision handling and export<br><br>Handles collisions and merges results for export. |
| `Merge & Export` | `n8n-nodes-base.code` | Export workflow | `Generate Stickies` | `Collision Detector` | ## Collision handling and export<br><br>Handles collisions and merges results for export. |
| `Collision Detector` | `n8n-nodes-base.code` | Detect collisions | `Merge & Export` | `Parse for analysis` | ## Collision handling and export<br><br>Handles collisions and merges results for export. |
| `Parse for analysis` | `n8n-nodes-base.code` | Parse for analysis | `Collision Detector` | `AI analysis` | ## Analyze workflow<br><br>Analyzes workflow using AI with given set of instructions |
| `AI analysis` | `@n8n/n8n-nodes-langchain.agent` | AI audit agent | `Parse for analysis`, `OpenAI Chat Model2`, `Structured Output Parser 2` | `Loop Over Items` | ## Analyze workflow<br><br>Analyzes workflow using AI with given set of instructions |
| `OpenAI Chat Model2` | `@n8n/n8n-nodes-langchain.lmChatOpenAi` | LLM service | None | `AI analysis`, `Structured Output Parser 2` | ## Analyze workflow<br><br>Analyzes workflow using AI with given set of instructions |
| `Structured Output Parser 2` | `@n8n/n8n-nodes-langchain.outputParserStructured` | Output parser | `OpenAI Chat Model2` | `AI analysis` | ## Analyze workflow<br><br>Analyzes workflow using AI with given set of instructions |
| `Filter` | `n8n-nodes-base.filter` | Filter payload | `Loop Over Items` | `Send a message` | ## Send analysis report to the Slack channel<br>We send the analysis to Slack channel, add reaction to it so that user doesn't have to look for it and upsert workflow information to the data table<br><br>We wait before we update rows because otherwise Slack Trigger would fire and mark workflows to not be scanned again |
| `Send a message` | `n8n-nodes-base.slack` | Send Slack report | `Filter` | `Add a reaction` | ## Send analysis report to the Slack channel<br>We send the analysis to Slack channel, add reaction to it so that user doesn't have to look for it and upsert workflow information to the data table<br><br>We wait before we update rows because otherwise Slack Trigger would fire and mark workflows to not be scanned again |
| `Add a reaction` | `n8n-nodes-base.slack` | Add Slack reaction | `Send a message` | `Wait` | ## Send analysis report to the Slack channel<br>We send the analysis to Slack channel, add reaction to it so that user doesn't have to look for it and upsert workflow information to the data table<br><br>We wait before we update rows because otherwise Slack Trigger would fire and mark workflows to not be scanned again |
| `Wait` | `n8n-nodes-base.wait` | Pause execution | `Add a reaction` | `Upsert row(s)` | ## Send analysis report to the Slack channel<br>We send the analysis to Slack channel, add reaction to it so that user doesn't have to look for it and upsert workflow information to the data table<br><br>We wait before we update rows because otherwise Slack Trigger would fire and mark workflows to not be scanned again |
| `Upsert row(s)` | `n8n-nodes-base.dataTable` | Upsert Data Table | `Wait` | None | ## Send analysis report to the Slack channel<br>We send the analysis to Slack channel, add reaction to it so that user doesn't have to look for it and upsert workflow information to the data table<br><br>We wait before we update rows because otherwise Slack Trigger would fire and mark workflows to not be scanned again |
| `Slack Trigger` | `n8n-nodes-base.slackTrigger` | Listen for reactions | None | `If reaction is :x:` | ## Listen for slack reactions<br>After :x: reaction is added, we check if the message id it was added to exists in the table, and if so, we disable scanning for this workflow |
| `If reaction is :x:` | `n8n-nodes-base.if` | Check reaction | `Slack Trigger` | `Switch` | ## Listen for slack reactions<br>After :x: reaction is added, we check if the message id it was added to exists in the table, and if so, we disable scanning for this workflow |
| `Switch` | `n8n-nodes-base.switch` | Route reaction type | `If reaction is :x:` | `Disable scan`, `Enable scan` | ## Listen for slack reactions<br>After :x: reaction is added, we check if the message id it was added to exists in the table, and if so, we disable scanning for this workflow |
| `Disable scan` | `n8n-nodes-base.dataTable` | Update scan status | `Switch` (Output 1) | `Notify about disabling scan` | ## Listen for slack reactions<br>After :x: reaction is added, we check if the message id it was added to exists in the table, and if so, we disable scanning for this workflow |
| `Enable scan` | `n8n-nodes-base.dataTable` | Update scan status | `Switch` (Output 2) | `Notify about enabling scan` | ## Listen for slack reactions<br>After :x: reaction is added, we check if the message id it was added to exists in the table, and if so, we disable scanning for this workflow |
| `Notify about disabling scan` | `n8n-nodes-base.slack` | Slack notification | `Disable scan` | None | ## Listen for slack reactions<br>After :x: reaction is added, we check if the message id it was added to exists in the table, and if so, we disable scanning for this workflow |
| `Notify about enabling scan` | `n8n-nodes-base.slack` | Slack notification | `Enable scan` | None | ## Listen for slack reactions<br>After :x: reaction is added, we check if the message id it was added to exists in the table, and if so, we disable scanning for this workflow |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Data Table Bootstrap Branch:**
   - Create a **Manual Trigger** node (`When clicking ‘Execute workflow’`).
   - Connect it to a **Data Table** node (`Create a data table`). Set resource to `Table`, operation to `Create`, table name to `AI Workflow Improvements`, and add columns: `workflowName` (string), `workflowId` (string), `scan` (boolean), `lastSlackMessageTimestampId` (string). Run this once manually to initialize the table.

2. **Create Workflow Fetch Branch:**
   - Create a **Schedule Trigger** node (`Schedule Trigger`). Configure interval to trigger every 7 days at 16:00.
   - Connect to an **n8n** node (`Get many workflows`). Configure with an **n8n API Key** credential and set resource to list workflows.
   - Connect `Get many workflows` to a **Data Table** node (`Get known workflows`). Set operation to `Get`, retrieve all, filtered where `scan` is `false`.
   - Connect both `Get many workflows` and `Get known workflows` to a **Merge** node (`Merge`). Set mode to `Combine`, advanced join mode to `keepNonMatches`, and match fields on `id` and `workflowId`.
   - Connect `Merge` to a **Split In Batches** node (`Loop Over Items`).

3. **Build Layout Documentation & AI Clustering Branch:**
   - From `Loop Over Items` (output batch branch), connect to a **Code** node (`Strip & Prepare`). Add JavaScript to strip sticky notes, extract AI sub-nodes, and track max slots.
   - Connect to a **Code** node (`Parse Nodes`) to extract node dimensions and context descriptions.
   - Connect to an **AI Agent** node (`AI Groups Logically`). Link an **OpenAI Chat Model** (`gpt-5.4-nano`) via `ai_languageModel` and a **Structured Output Parser** (`Structured Output Parser`) via `ai_outputParser`. Configure API credentials (`OpenAi Api Key`).
   - Chain sequential **Code** nodes:
     - `Compute Bounding Boxes`: Calculates canvas clusters.
     - `Collision Resolution`: Iteratively adjusts overlapping bounding boxes.
     - `Generate Stickies`: Generates overview and group sticky notes with UUIDs.
     - `Merge & Export`: Replaces workflow nodes with updated stickies and shifted coordinates.
     - `Collision Detector`: Validates that no foreign nodes overlap sticky boundaries.

4. **Build AI Analysis & Slack Reporting Branch:**
   - From `Collision Detector`, connect to a **Code** node (`Parse for analysis`) to prepare the clean workflow structure for evaluation.
   - Connect to an **AI Agent** node (`AI analysis`). Link an **OpenAI Chat Model** (`gpt-4o-mini`) via `ai_languageModel` and a **Structured Output Parser** (`Structured Output Parser 2`) via `ai_outputParser`.
   - Connect `AI analysis` back to `Loop Over Items` to continue batch iteration.
   - From `Loop Over Items` (output results branch), connect to a **Filter** node (`Filter`) to check that the evaluation object is not empty.
   - Connect to a **Slack** action node (`Send a message`). Configure Slack OAuth2 credentials, set message type to `Block Kit`, target channel `workflow-improvements`, and paste rich block JSON templates.
   - Connect to a Slack action node (`Add a reaction`). Set operation to `Reaction`, action to `Add`, name to `x`, channel to `={{ $json.channel }}`, and timestamp to `={{ $json.message_timestamp }}`.
   - Connect to a **Wait** node (`Wait`) with default settings.
   - Connect to a **Data Table** node (`Upsert row(s)`). Set operation to `Upsert`, match on `workflowId`, and map fields for `workflowId`, `workflowName`, `scan` (`true`), and `lastSlackMessageTimestampId`.

5. **Build Slack Reaction Event Handler Branch:**
   - Create a **Slack Trigger** node (`Slack Trigger`). Configure Slack OAuth2 credentials, event type `any_event`, and monitor channel `workflow-improvements`.
   - Connect to an **If** node (`If reaction is :x:`). Condition: `={{ $json.reaction }}` equals `x`.
   - Connect to a **Switch** node (`Switch`). Configure rules for `reaction_added` (output 0) and `reaction_removed` (output 1).
   - Connect output 0 to a **Data Table** node (`Disable scan`). Set operation to `Update`, filter where `lastSlackMessageTimestampId` equals `={{ $json.item.ts }}`, and set `scan` to `false`. Connect to a **Slack** node (`Notify about disabling scan`) to post confirmation.
   - Connect output 1 to a **Data Table** node (`Enable scan`). Set operation to `Update`, filter where `lastSlackMessageTimestampId` equals `={{ $json.item.ts }}`, and set `scan` to `true`. Connect to a **Slack** node (`Notify about enabling scan`) to post confirmation.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Need help? Contact us or visit our website. | Contact link: [sailingbyte.com/contact/](https://sailingbyte.com/contact/) |
| Visit Sailing Byte website. | Website link: [sailingbyte.com](https://sailingbyte.com) |
| n8n Error Handling Documentation | [n8n Docs - Error Handling](https://docs.n8n.io/flow-logic/error-handling/) |
| n8n Sub-workflows Documentation | [n8n Docs - Execute Workflow](https://docs.n8n.io/flow-logic/execute-workflow/) |
| n8n Expressions Documentation | [n8n Docs - Expressions](https://docs.n8n.io/code/expressions/) |
| n8n Credentials Documentation | [n8n Docs - Credentials](https://docs.n8n.io/credentials/) |
| Community Best Practices | [n8n Community Forum](https://community.n8n.io/t/best-practices-for-structuring-n8n-workflows-for-scale-and-long-term-maintainability/248671) |
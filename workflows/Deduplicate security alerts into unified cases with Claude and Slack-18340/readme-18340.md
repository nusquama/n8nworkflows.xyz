Deduplicate security alerts into unified cases with Claude and Slack

https://n8nworkflows.xyz/workflows/deduplicate-security-alerts-into-unified-cases-with-claude-and-slack-18340


# Deduplicate security alerts into unified cases with Claude and Slack

### 1. Workflow Overview

The "AI Security Alert Deduplication Engine" workflow ingests security alerts from multiple tools (SIEM, EDR, and Cloud), normalizes them into a canonical format, and buffers them for batch processing. On a scheduled interval, it pulls buffered alerts, groups them into fingerprints, separates clear duplicates from ambiguous ones, utilizes Anthropic Claude to semantically cluster the ambiguous group, constructs unified security cases, syncs them to an external ticketing system via a bulk upsert, and sends a summary notification to Slack.

The workflow logic is divided into the following functional blocks:
- **1.1 Alert Intake & Normalization:** Receives webhooks from SIEM, EDR, and Cloud platforms and maps raw payloads to a shared schema.
- **1.2 Alert Buffering:** Combines normalized alerts and pushes them to an external buffer service.
- **1.3 Batch Fetching & Pre-Clustering:** Periodically fetches buffered alerts for the processing window and groups them based on host, rule, severity, and temporal proximity.
- **1.4 AI Semantic Clustering:** Evaluates ambiguous clusters using Anthropic Claude to determine if alerts represent duplicate or separate incidents.
- **1.5 Case Construction & Notification:** Merges all resolved clusters, builds unified security cases, updates the ticketing system, and posts status updates to Slack.

---

### 2. Block-by-Block Analysis

#### 2.1 Alert Intake & Normalization
- **Overview:** Receives raw security events via three separate HTTP webhooks, normalizes each schema into a unified format, and funnels them into a single merge node.
- **Nodes Involved:** 
  - `When SIEM Alert Received`
  - `When EDR Alert Received`
  - `When Cloud Alert Received`
  - `Normalize SIEM Alert`
  - `Normalize EDR Alert`
  - `Normalize Cloud Alert`
  - `Merge Normalized Alerts`
- **Node Details:**
  - `When SIEM Alert Received`
    - **Type:** `n8n-nodes-base.webhook` (v2)
    - **Role:** Entry point for SIEM alerts.
    - **Configuration:** HTTP Method `POST`, Path `alerts/siem`.
    - **Input / Output:** Output connects to `Normalize SIEM Alert`.
    - **Failure Risks:** Webhook endpoint down or invalid payload format.
  - `When EDR Alert Received`
    - **Type:** `n8n-nodes-base.webhook` (v2)
    - **Role:** Entry point for EDR alerts.
    - **Configuration:** HTTP Method `POST`, Path `alerts/edr`.
    - **Input / Output:** Output connects to `Normalize EDR Alert`.
  - `When Cloud Alert Received`
    - **Type:** `n8n-nodes-base.webhook` (v2)
    - **Role:** Entry point for cloud security alerts.
    - **Configuration:** HTTP Method `POST`, Path `alerts/cloud`.
    - **Input / Output:** Output connects to `Normalize Cloud Alert`.
  - `Normalize SIEM Alert`
    - **Type:** `n8n-nodes-base.code` (v2)
    - **Role:** Maps raw SIEM properties (`alertId`, `ruleName`, etc.) to canonical schema.
    - **Configuration:** `runOnceForEachItem` mode with custom JavaScript.
    - **Input / Output:** Input from SIEM webhook; output to input index 0 of `Merge Normalized Alerts`.
  - `Normalize EDR Alert`
    - **Type:** `n8n-nodes-base.code` (v2)
    - **Role:** Maps raw EDR properties (`detectionId`, `deviceName`, etc.) to canonical schema.
    - **Configuration:** `runOnceForEachItem` mode with custom JavaScript.
    - **Input / Output:** Input from EDR webhook; output to input index 1 of `Merge Normalized Alerts`.
  - `Normalize Cloud Alert`
    - **Type:** `n8n-nodes-base.code` (v2)
    - **Role:** Maps raw cloud finding properties (`findingId`, `resourceId`, etc.) to canonical schema.
    - **Configuration:** `runOnceForEachItem` mode with custom JavaScript.
    - **Input / Output:** Input from Cloud webhook; output to input index 2 of `Merge Normalized Alerts`.
  - `Merge Normalized Alerts`
    - **Type:** `n8n-nodes-base.merge` (v3.2)
    - **Role:** Combines the three normalization streams into one.
    - **Configuration:** `numberInputs: 3`.
    - **Input / Output:** Inputs from the three normalization nodes; output to `Post Alert to Buffer`.

---

#### 2.2 Alert Buffering
- **Overview:** Sends each normalized alert individually to an external storage service via HTTP POST to preserve alerts for batch deduplication.
- **Nodes Involved:**
  - `Post Alert to Buffer`
- **Node Details:**
  - `Post Alert to Buffer`
    - **Type:** `n8n-nodes-base.httpRequest` (v4.2)
    - **Role:** Posts normalized alerts to the external buffering endpoint.
    - **Configuration:** Method `POST`, URL `https://api.alert-buffer.example.com/v1/alerts`, authentication set to HTTP Header Auth. JSON body configured as `={{ JSON.stringify($json) }}`.
    - **Input / Output:** Input from `Merge Normalized Alerts`; output is standalone (buffer sink).
    - **Failure Risks:** HTTP 4xx/5xx errors from the buffer API, network timeout, or authentication failure.

---

#### 2.3 Batch Fetching & Pre-Clustering
- **Overview:** Triggers on a schedule to fetch accumulated alerts from the buffer, computes strict fingerprints, and divides alerts into clear groups or ambiguous sets.
- **Nodes Involved:**
  - `Every Batch Processing Window`
  - `Fetch Buffered Alerts`
  - `Pre-Cluster Alerts`
  - `Check for Ambiguous Clusters`
- **Node Details:**
  - `Every Batch Processing Window`
    - **Type:** `n8n-nodes-base.scheduleTrigger` (v1.2)
    - **Role:** Triggers deduplication batches periodically.
    - **Configuration:** Interval set to minutes.
    - **Input / Output:** Output connects to `Fetch Buffered Alerts`.
  - `Fetch Buffered Alerts`
    - **Type:** `n8n-nodes-base.httpRequest` (v4.2)
    - **Role:** Retrieves buffered alerts for the processing window.
    - **Configuration:** Method `GET`, URL `https://api.alert-buffer.example.com/v1/alerts/window`, HTTP Header Auth.
    - **Input / Output:** Input from schedule trigger; output to `Pre-Cluster Alerts`.
  - `Pre-Cluster Alerts`
    - **Type:** `n8n-nodes-base.code` (v2)
    - **Role:** Groups alerts by host, rule ID, and severity, then evaluates time thresholds (5 minutes) to separate obvious duplicates from ambiguous alerts.
    - **Configuration:** Custom JavaScript parsing response and returning `clearClusters`, `ambiguousClusters`, and `hasAmbiguous` boolean.
    - **Input / Output:** Input from fetch request; output to `Check for Ambiguous Clusters`.
  - `Check for Ambiguous Clusters`
    - **Type:** `n8n-nodes-base.if` (v2.2)
    - **Role:** Routes execution depending on whether ambiguous clusters exist.
    - **Configuration:** Condition checks if `={{ $json.hasAmbiguous }}` equals `true`.
    - **Input / Output:** Input from pre-cluster code; True branch outputs to `AI Semantic Clustering Agent`, False branch outputs to `Pass Clear Clusters`.

---

#### 2.4 AI Semantic Clustering
- **Overview:** Uses an Anthropic Claude model to evaluate ambiguous alert clusters and decides whether they should be merged or kept separate, applying a parsing safety fallback.
- **Nodes Involved:**
  - `AI Semantic Clustering Agent`
  - `Claude Clustering Model`
  - `Apply AI Cluster Resolution`
- **Node Details:**
  - `AI Semantic Clustering Agent`
    - **Type:** `@n8n/n8n-nodes-langchain.agent` (v1.6)
    - **Role:** Advanced AI agent parsing ambiguous alert data to output structured JSON clustering decisions.
    - **Configuration:** Prompt type defined with a strict system prompt instructing JSON-only responses. Connected to Claude language model.
    - **Input / Output:** Input from `Check for Ambiguous Clusters` (True branch); output to `Apply AI Cluster Resolution`.
    - **Failure Risks:** LLM output format deviation or token limits exceeded.
  - `Claude Clustering Model`
    - **Type:** `@n8n/n8n-nodes-langchain.lmChatAnthropic` (v1.3)
    - **Role:** LLM sub-node providing Anthropic integration.
    - **Configuration:** Model set to `claude-sonnet-4-20250514`, temperature `0.1`.
    - **Input / Output:** Connected to `AI Semantic Clustering Agent` via `ai_languageModel` hook.
    - **Credential Requirement:** Anthropic API key.
  - `Apply AI Cluster Resolution`
    - **Type:** `n8n-nodes-base.code` (v2)
    - **Role:** Parses LLM string response, strips markdown formatting, and merges AI decisions with the cluster dataset. Includes fallback logic to treat ambiguous items as separate cases if parsing fails.
    - **Configuration:** `runOnceForEachItem` with custom JavaScript.
    - **Input / Output:** Input from AI Agent; output to `Merge Cluster Results`.

---

#### 2.5 Case Construction & Notification
- **Overview:** Recombines clear and AI-resolved clusters, aggregates severity ranks, builds unified cases, bulk upserts them into a ticketing system, and posts a notification to Slack.
- **Nodes Involved:**
  - `Pass Clear Clusters`
  - `Merge Cluster Results`
  - `Build Security Cases`
  - `Post to Ticket System API`
  - `Send Slack Notification`
- **Node Details:**
  - `Pass Clear Clusters`
    - **Type:** `n8n-nodes-base.code` (v2)
    - **Role:** Formats clear pre-clusters to match the structure expected by the downstream merge node.
    - **Configuration:** `runOnceForEachItem` with custom JavaScript.
    - **Input / Output:** Input from `Check for Ambiguous Clusters` (False branch); output to input index 1 of `Merge Cluster Results`.
  - `Merge Cluster Results`
    - **Type:** `n8n-nodes-base.merge` (v3.2)
    - **Role:** Unifies AI-resolved clusters and clear clusters into a single stream.
    - **Configuration:** Standard multi-input merge.
    - **Input / Output:** Inputs from `Apply AI Cluster Resolution` and `Pass Clear Clusters`; output to `Build Security Cases`.
  - `Build Security Cases`
    - **Type:** `n8n-nodes-base.code` (v2)
    - **Role:** Aggregates alerts inside each cluster into a single security case object, determines highest severity, compiles source lists, and extracts member IDs.
    - **Configuration:** Custom JavaScript calculating severity ranks (`critical: 4`, `high: 3`, `medium: 2`, `low: 1`).
    - **Input / Output:** Input from merge results; output to `Post to Ticket System API`.
  - `Post to Ticket System API`
    - **Type:** `n8n-nodes-base.httpRequest` (v4.2)
    - **Role:** Performs bulk upsert of unified security cases into the ticketing system.
    - **Configuration:** Method `POST`, URL `https://api.ticketing-system.example.com/v1/cases/bulk-upsert`, HTTP Header Auth. Body sends JSON-encoded cases array.
    - **Input / Output:** Input from case builder; output to `Send Slack Notification`.
    - **Failure Risks:** Ticketing API downtime, validation errors on case payload.
  - `Send Slack Notification`
    - **Type:** `n8n-nodes-base.slack` (v2.2)
    - **Role:** Sends an update message summarizing the processing cycle and case counts to a Slack channel.
    - **Configuration:** Resource `channel`, channel set to `security-alerts`. Message template includes window timestamp and total cases created or updated.
    - **Input / Output:** Input from ticketing HTTP request; terminal node.
    - **Credential Requirement:** Slack OAuth2 / Bot token.
    - **Failure Risks:** Invalid channel ID, insufficient bot permissions to post in target channel.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Sticky Note` | `n8n-nodes-base.stickyNote` | Documentation and setup instructions | None | None | ## AI Security Alert Deduplication Engine... |
| `Sticky Note1` | `n8n-nodes-base.stickyNote` | SIEM intake documentation | None | None | ## SIEM alert intake... |
| `Sticky Note2` | `n8n-nodes-base.stickyNote` | EDR intake documentation | None | None | ## EDR alert intake... |
| `Sticky Note3` | `n8n-nodes-base.stickyNote` | Cloud intake documentation | None | None | ## Cloud alert intake... |
| `Sticky Note4` | `n8n-nodes-base.stickyNote` | Buffer integration documentation | None | None | ## Buffer normalized alerts... |
| `Sticky Note5` | `n8n-nodes-base.stickyNote` | Batch fetch documentation | None | None | ## Fetch batch window... |
| `Sticky Note6` | `n8n-nodes-base.stickyNote` | Fingerprinting documentation | None | None | ## Fingerprint and route... |
| `Sticky Note7` | `n8n-nodes-base.stickyNote` | AI cluster resolution documentation | None | None | ## Resolve ambiguous clusters... |
| `Sticky Note8` | `n8n-nodes-base.stickyNote` | Clear cluster path documentation | None | None | ## Pass clear clusters... |
| `Sticky Note9` | `n8n-nodes-base.stickyNote` | Cluster merge documentation | None | None | ## Merge cluster outputs... |
| `Sticky Note10` | `n8n-nodes-base.stickyNote` | Ticketing and Slack notification docs | None | None | ## Create cases and notify... |
| `When SIEM Alert Received` | `n8n-nodes-base.webhook` | Webhook receiver for SIEM alerts | None | `Normalize SIEM Alert` | ## SIEM alert intake... |
| `When EDR Alert Received` | `n8n-nodes-base.webhook` | Webhook receiver for EDR alerts | None | `Normalize EDR Alert` | ## EDR alert intake... |
| `When Cloud Alert Received` | `n8n-nodes-base.webhook` | Webhook receiver for Cloud alerts | None | `Normalize Cloud Alert` | ## Cloud alert intake... |
| `Normalize SIEM Alert` | `n8n-nodes-base.code` | Normalizes SIEM payload structure | `When SIEM Alert Received` | `Merge Normalized Alerts` | ## SIEM alert intake... |
| `Normalize EDR Alert` | `n8n-nodes-base.code` | Normalizes EDR payload structure | `When EDR Alert Received` | `Merge Normalized Alerts` | ## EDR alert intake... |
| `Normalize Cloud Alert` | `n8n-nodes-base.code` | Normalizes Cloud payload structure | `When Cloud Alert Received` | `Merge Normalized Alerts` | ## Cloud alert intake... |
| `Merge Normalized Alerts` | `n8n-nodes-base.merge` | Merges three normalization data streams | `Normalize SIEM Alert`, `Normalize EDR Alert`, `Normalize Cloud Alert` | `Post Alert to Buffer` | ## Buffer normalized alerts... |
| `Post Alert to Buffer` | `n8n-nodes-base.httpRequest` | Sends normalized alert to buffer API | `Merge Normalized Alerts` | None | ## Buffer normalized alerts... |
| `Every Batch Processing Window` | `n8n-nodes-base.scheduleTrigger` | Triggers periodic batch cycle | None | `Fetch Buffered Alerts` | ## Fetch batch window... |
| `Fetch Buffered Alerts` | `n8n-nodes-base.httpRequest` | Retrieves buffered alerts for time window | `Every Batch Processing Window` | `Pre-Cluster Alerts` | ## Fetch batch window... |
| `Pre-Cluster Alerts` | `n8n-nodes-base.code` | Fingerprints and splits clear/ambiguous alerts | `Fetch Buffered Alerts` | `Check for Ambiguous Clusters` | ## Fingerprint and route... |
| `Check for Ambiguous Clusters` | `n8n-nodes-base.if` | Routes workflow based on ambiguity presence | `Pre-Cluster Alerts` | `AI Semantic Clustering Agent`, `Pass Clear Clusters` | ## Fingerprint and route... |
| `AI Semantic Clustering Agent` | `@n8n/n8n-nodes-langchain.agent` | AI agent analyzing ambiguous alerts | `Check for Ambiguous Clusters` | `Apply AI Cluster Resolution` | ## Resolve ambiguous clusters... |
| `Claude Clustering Model` | `@n8n/n8n-nodes-langchain.lmChatAnthropic` | Provides Claude LLM model configuration | `Claude Clustering Model` (AI link) | `AI Semantic Clustering Agent` | ## Resolve ambiguous clusters... |
| `Apply AI Cluster Resolution` | `n8n-nodes-base.code` | Parses AI response and finalizes clusters | `AI Semantic Clustering Agent` | `Merge Cluster Results` | ## Resolve ambiguous clusters... |
| `Pass Clear Clusters` | `n8n-nodes-base.code` | Prepares unambiguous clusters for merge | `Check for Ambiguous Clusters` | `Merge Cluster Results` | ## Pass clear clusters... |
| `Merge Cluster Results` | `n8n-nodes-base.merge` | Recombines clear and AI-resolved clusters | `Apply AI Cluster Resolution`, `Pass Clear Clusters` | `Build Security Cases` | ## Merge cluster outputs... |
| `Build Security Cases` | `n8n-nodes-base.code` | Compiles clusters into unified cases | `Merge Cluster Results` | `Post to Ticket System API` | ## Create cases and notify... |
| `Post to Ticket System API` | `n8n-nodes-base.httpRequest` | Bulk upserts unified cases into ticketing | `Build Security Cases` | `Send Slack Notification` | ## Create cases and notify... |
| `Send Slack Notification` | `n8n-nodes-base.slack` | Posts cycle summary to Slack channel | `Post to Ticket System API` | None | ## Create cases and notify... |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Intake Webhooks:**
   - Add a **Webhook** node named `When SIEM Alert Received`, set method to `POST`, and path to `alerts/siem`.
   - Add a **Webhook** node named `When EDR Alert Received`, set method to `POST`, and path to `alerts/edr`.
   - Add a **Webhook** node named `When Cloud Alert Received`, set method to `POST`, and path to `alerts/cloud`.

2. **Add Normalization Code Nodes:**
   - Add a **Code** node named `Normalize SIEM Alert` (Mode: *Run Once for Each Item*), paste the SIEM normalization JavaScript snippet, and connect `When SIEM Alert Received` to it.
   - Add a **Code** node named `Normalize EDR Alert` (Mode: *Run Once for Each Item*), paste the EDR normalization JavaScript snippet, and connect `When EDR Alert Received` to it.
   - Add a **Code** node named `Normalize Cloud Alert` (Mode: *Run Once for Each Item*), paste the Cloud normalization JavaScript snippet, and connect `When Cloud Alert Received` to it.

3. **Buffer Integration:**
   - Add a **Merge** node named `Normalize Normalized Alerts` configured with `3` inputs. Connect the three normalization nodes to inputs 0, 1, and 2 respectively.
   - Add an **HTTP Request** node named `Post Alert to Buffer`:
     - Method: `POST`
     - URL: `https://api.alert-buffer.example.com/v1/alerts`
     - Authentication: HTTP Header Auth (configure credentials).
     - Body: Send JSON `={{ JSON.stringify($json) }}`.
     - Connect `Normalize Normalized Alerts` output to this node.

4. **Scheduled Batch Processing:**
   - Add a **Schedule Trigger** node named `Every Batch Processing Window` and set the interval rule to run every minute (or desired interval).
   - Add an **HTTP Request** node named `Fetch Buffered Alerts`:
     - Method: `GET`
     - URL: `https://api.alert-buffer.example.com/v1/alerts/window`
     - Authentication: HTTP Header Auth.
     - Connect `Every Batch Processing Window` to this node.

5. **Pre-Clustering & Branching:**
   - Add a **Code** node named `Pre-Cluster Alerts`, paste the fingerprint and time-window evaluation script, and connect `Fetch Buffered Alerts` to it.
   - Add an **If** node named `Check for Ambiguous Clusters`:
     - Condition: `={{ $json.hasAmbiguous }}` equals `true`.
     - Connect `Pre-Cluster Alerts` output to this node.

6. **AI Resolution Branch:**
   - Add an **Advanced AI Agent** node named `AI Semantic Clustering Agent` (Prompt type: *Define*). Connect the *True* output of `Check for Ambiguous Clusters` to it.
   - Add a **Claude Chat Model** node named `Claude Clustering Model`:
     - Model: `claude-sonnet-4-20250514`
     - Temperature: `0.1`
     - Configure Anthropic API credentials.
     - Connect its output to the `ai_languageModel` input of the AI Agent.
   - Add a **Code** node named `Apply AI Cluster Resolution`, paste the JSON parsing and safety fallback script, and connect the output of the AI Agent to it.

7. **Clear Cluster Branch:**
   - Add a **Code** node named `Pass Clear Clusters`, paste the clear cluster formatting script, and connect the *False* output of `Check for Ambiguous Clusters` to it.

8. **Merge & Case Construction:**
   - Add a **Merge** node named `Merge Cluster Results` (default inputs). Connect `Apply AI Cluster Resolution` and `Pass Clear Clusters` to its inputs.
   - Add a **Code** node named `Build Security Cases`, paste the case aggregation and severity ranking script, and connect `Merge Cluster Results` to it.

9. **Sync & Notification:**
   - Add an **HTTP Request** node named `Post to Ticket System API`:
     - Method: `POST`
     - URL: `https://api.ticketing-system.example.com/v1/cases/bulk-upsert`
     - Authentication: HTTP Header Auth.
     - Body: Send JSON `={{ JSON.stringify({ cases: $json.cases }) }}`.
     - Connect `Build Security Cases` to this node.
   - Add a **Slack** node named `Send Slack Notification`:
     - Resource: `Channel`
     - Channel ID: `security-alerts`
     - Text: `Security alert cycle {{ $json.windowTimestamp }}: {{ $json.cases.length }} unified case(s) created/updated. Review in the ticketing system.`
     - Configure Slack Bot/OAuth2 credentials.
     - Connect `Post to Ticket System API` to this node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| AI Security Alert Deduplication Engine | Workflow purpose and high-level architecture overview |
| External Buffer Service API | Used for temporary event storage between webhook intake and scheduled batch processing (`https://api.alert-buffer.example.com/v1/alerts`) |
| Ticketing System API | Target endpoint for bulk upserting unified security cases (`https://api.ticketing-system.example.com/v1/cases/bulk-upsert`) |
| Anthropic Claude Model | Utilized via LangChain integration for semantic clustering of ambiguous alert groups (`claude-sonnet-4-20250514`) |
| Slack Integration | Posts real-time cycle summaries and case counts to the `#security-alerts` channel |
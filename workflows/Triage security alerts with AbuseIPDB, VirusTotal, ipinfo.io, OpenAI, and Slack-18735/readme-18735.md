Triage security alerts with AbuseIPDB, VirusTotal, ipinfo.io, OpenAI, and Slack

https://n8nworkflows.xyz/workflows/triage-security-alerts-with-abuseipdb--virustotal--ipinfo-io--openai--and-slack-18735


# Triage security alerts with AbuseIPDB, VirusTotal, ipinfo.io, OpenAI, and Slack

### 1. Workflow Overview

The **Security Alert Triage & Enrichment** workflow automates the ingestion, contextual enrichment, artificial intelligence-driven analysis, and multi-channel routing of security alerts. Designed for Security Operations Centers (SOC), it removes manual overhead by instantly gathering threat intelligence on incoming indicators of compromise (IoCs), generating structured security assessments, and escalating critical incidents automatically.

The logic is grouped into five functional blocks:
- **1.1 Input Reception & Normalization:** Captures inbound security payloads via HTTP webhook and standardizes them into a uniform schema.
- **1.2 Threat Intelligence Enrichment:** Queries external threat feeds (AbuseIPDB, ipinfo.io, and VirusTotal) to gather reputation metrics, geolocation, ASN ownership, and file hash analysis.
- **1.3 AI-Driven Security Triage:** Feeds the consolidated enrichment bundle into an OpenAI-powered agent to produce a SOC-style analysis containing severity scores, false-positive likelihoods, summaries, and recommended remediations.
- **1.4 Response Dispatch & Webhook Return:** Responds immediately to the original webhook caller with the triage payload.
- **1.5 Severity Routing & Escalation:** Evaluates the severity level, routing high/critical alerts to an urgent Slack channel and a ticketing endpoint, while routing routine alerts to a logging channel.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Normalization
- **Overview:** Receives raw security alerts via HTTP POST and transforms disparate payloads into a clean, consistent JSON schema for downstream processing.
- **Nodes Involved:** `When Security Alert Posted`, `Normalize Security Alert`
- **Node Details:**
  - **When Security Alert Posted**
    - *Type and Technical Role:* `n8n-nodes-base.webhook` (Webhook Trigger). Acts as the entry point, listening for incoming HTTP POST requests at path `/security-alert`.
    - *Configuration Choices:* Standard webhook trigger configured with execution order v1.
    - *Key Expressions or Variables:* Uses webhook ID `d2a17dae-bba5-4f55-a8d6-fb262dcb4da2`.
    - *Input and Output Connections:* Input: External HTTP POST. Output: Connects to `Normalize Security Alert`.
    - *Version-Specific Requirements:* Version 2.
    - *Edge Cases or Potential Failure Types:* Malformed JSON payloads, network timeouts, unauthenticated public endpoints.
  - **Normalize Security Alert**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript/Python Code Execution). Extracts and standardizes common fields such as alert IDs, IP addresses, hosts, users, rule names, severity, and file hashes.
    - *Configuration Choices:* Executes custom transformation logic on incoming binary or JSON payload items.
    - *Key Expressions or Variables:* Reads properties from `{{ $json }}`.
    - *Input and Output Connections:* Input: `When Security Alert Posted`. Output: Connects to `Fetch AbuseIPDB Data`.
    - *Version-Specific Requirements:* Version 2.
    - *Edge Cases or Potential Failure Types:* Missing mandatory keys (e.g., source IP or file hash) resulting in undefined properties.

#### 1.2 Threat Intelligence Enrichment
- **Overview:** Sequentially queries multiple third-party intelligence APIs to enrich the normalized alert data with reputation scores, geolocation context, and file analysis.
- **Nodes Involved:** `Fetch AbuseIPDB Data`, `Fetch IP Geo Information`, `Fetch VirusTotal File Data`, `Consolidate Data Enrichment`
- **Node Details:**
  - **Fetch AbuseIPDB Data**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (HTTP Request). Queries AbuseIPDB to retrieve IP reputation and abuse confidence scores.
    - *Configuration Choices:* GET/POST request configured with appropriate API headers and URL parameters referencing the normalized IP.
    - *Key Expressions or Variables:* References the normalized source IP from the preceding code node.
    - *Input and Output Connections:* Input: `Normalize Security Alert`. Output: Connects to `Fetch IP Geo Information`.
    - *Version-Specific Requirements:* Version 4.4.
    - *Edge Cases or Potential Failure Types:* API rate limits, invalid API keys, or timeouts from the AbuseIPDB service.
  - **Fetch IP Geo Information**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (HTTP Request). Queries `ipinfo.io` to retrieve geographic location, organization, and Autonomous System Number (ASN) details.
    - *Configuration Choices:* HTTP request targeting the ipinfo.io API endpoint.
    - *Key Expressions or Variables:* References the normalized IP address.
    - *Input and Output Connections:* Input: `Fetch AbuseIPDB Data`. Output: Connects to `Fetch VirusTotal File Data`.
    - *Version-Specific Requirements:* Version 4.4.
    - *Edge Cases or Potential Failure Types:* Unreachable lookup endpoint, expired access tokens.
  - **Fetch VirusTotal File Data**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (HTTP Request). Queries VirusTotal for analysis reports related to the extracted file hash.
    - *Configuration Choices:* HTTP request using VirusTotal API headers and the normalized file hash parameter.
    - *Key Expressions or Variables:* References the normalized file hash value.
    - *Input and Output Connections:* Input: `Fetch IP Geo Information`. Output: Connects to `Consolidate Data Enrichment`.
    - *Version-Specific Requirements:* Version 4.4.
    - *Edge Cases or Potential Failure Types:* Hash not found in VirusTotal database (404 response), API throttling.
  - **Consolidate Data Enrichment**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript/Python Code Execution). Merges the responses from AbuseIPDB, ipinfo.io, and VirusTotal into a unified data structure.
    - *Configuration Choices:* Executes aggregation logic across preceding node outputs.
    - *Key Expressions or Variables:* Aggregates data streams using internal execution references.
    - *Input and Output Connections:* Input: `Fetch VirusTotal File Data`. Output: Connects to `Security Triage Agent`.
    - *Version-Specific Requirements:* Version 2.
    - *Edge Cases or Potential Failure Types:* Partial API failures resulting in missing enrichment blocks.

#### 1.3 AI-Driven Security Triage
- **Overview:** Leverages an OpenAI language model agent to analyze the enriched security alert and produce a structured triage output.
- **Nodes Involved:** `Security Triage Agent`, `OpenAI Security Model`
- **Node Details:**
  - **Security Triage Agent**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.agent` (LangChain AI Agent). Acts as the orchestration layer for prompt execution using the connected language model.
    - *Configuration Choices:* Configured to interpret the consolidated enrichment data and generate SOC triage outputs (severity, false-positive likelihood, summary, recommended actions, and confidence score).
    - *Key Expressions or Variables:* Ingests consolidated alert and enrichment data as context.
    - *Input and Output Connections:* Input: `Consolidate Data Enrichment`. Output: Connects to `Parse Triage Results`. Linked to `OpenAI Security Model` via AI language model connection.
    - *Version-Specific Requirements:* Version 3.1.
    - *Edge Cases or Potential Failure Types:* LLM service outages, malformed JSON responses from the model, or token limit overages.
  - **OpenAI Security Model**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.lmChatOpenAi` (OpenAI Chat Model). Provides the underlying Large Language Model capabilities for the agent.
    - *Configuration Choices:* Configured with an OpenAI API credential and model parameters.
    - *Key Expressions or Variables:* Uses OpenAI API credentials.
    - *Input and Output Connections:* Output: Connects via AI language model wire to `Security Triage Agent`.
    - *Version-Specific Requirements:* Version 1.3.
    - *Edge Cases or Potential Failure Types:* Authentication errors, insufficient OpenAI API quota.

#### 1.4 Response Dispatch & Webhook Return
- **Overview:** Parses the AI agent's triage output and returns the final structured assessment back to the original webhook caller.
- **Nodes Involved:** `Parse Triage Results`, `Return Webhook Response`
- **Node Details:**
  - **Parse Triage Results**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript/Python Code Execution). Cleans and parses the text output from the AI agent into strict JSON format.
    - *Configuration Choices:* Executes parsing logic on the model response text.
    - *Key Expressions or Variables:* References the output string from `Security Triage Agent`.
    - *Input and Output Connections:* Input: `Security Triage Agent`. Output: Connects concurrently to `Return Webhook Response` and `Check Severity Level`.
    - *Version-Specific Requirements:* Version 2.
    - *Edge Cases or Potential Failure Types:* JSON parsing errors if the LLM output contains invalid formatting or markdown wrappers.
  - **Return Webhook Response**
    - *Type and Technical Role:* `n8n-nodes-base.respondToWebhook` (Respond to Webhook). Sends the final enriched alert and triage JSON payload back to the synchronous HTTP webhook requester.
    - *Configuration Choices:* Configured to return response data immediately.
    - *Key Expressions or Variables:* References the parsed triage results.
    - *Input and Output Connections:* Input: `Parse Triage Results`. Output: None (Terminal node for webhook response).
    - *Version-Specific Requirements:* Version 1.1.
    - *Edge Cases or Potential Failure Types:* Connection closed by client before response is sent.

#### 1.5 Severity Routing & Escalation
- **Overview:** Evaluates the triage severity level to route urgent threats to incident response channels while logging routine events.
- **Nodes Involved:** `Check Severity Level`, `Post to Slack Urgent Channel`, `Send Incident Ticket`, `Post to Slack Routine Log`
- **Node Details:**
  - **Check Severity Level**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Conditional Branching). Inspects the severity determination from the parsed triage output.
    - *Configuration Choices:* Conditional rules evaluating severity levels (e.g., High/Critical vs. Medium/Low).
    - *Key Expressions or Variables:* Evaluates severity property from `Parse Triage Results`.
    - *Input and Output Connections:* Input: `Parse Triage Results`. Output: True branch connects to `Post to Slack Urgent Channel`; False branch connects to `Post to Slack Routine Log`.
    - *Version-Specific Requirements:* Version 2.2.
    - *Edge Cases or Potential Failure Types:* Unrecognized severity string values causing unexpected routing.
  - **Post to Slack Urgent Channel**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (HTTP Request). Posts an urgent notification to a designated Slack incoming webhook channel.
    - *Configuration Choices:* HTTP POST request with payload formatted for Slack incoming webhooks.
    - *Key Expressions or Variables:* References high-severity triage details.
    - *Input and Output Connections:* Input: `Check Severity Level` (True). Output: Connects to `Send Incident Ticket`.
    - *Version-Specific Requirements:* Version 4.4.
    - *Edge Cases or Potential Failure Types:* Slack webhook URL deprecation or rate-limiting.
  - **Send Incident Ticket**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (HTTP Request). Automatically creates an incident ticket by submitting a payload to a configured ticketing system endpoint.
    - *Configuration Choices:* HTTP POST request targeting the enterprise ticketing API.
    - *Key Expressions or Variables:* References enriched alert and triage data.
    - *Input and Output Connections:* Input: `Post to Slack Urgent Channel`. Output: None.
    - *Version-Specific Requirements:* Version 4.4.
    - *Edge Cases or Potential Failure Types:* Ticketing endpoint unavailability, authentication failure.
  - **Post to Slack Routine Log**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (HTTP Request). Logs routine or low-severity alerts to a secondary Slack channel.
    - *Configuration Choices:* HTTP POST request using a routine Slack incoming webhook URL.
    - *Key Expressions or Variables:* References routine triage details.
    - *Input and Output Connections:* Input: `Check Severity Level` (False). Output: None.
    - *Version-Specific Requirements:* Version 4.4.
    - *Edge Cases or Potential Failure Types:* Network timeout connecting to Slack.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| When Security Alert Posted | `n8n-nodes-base.webhook` | Webhook Trigger | None | Normalize Security Alert | |
| Normalize Security Alert | `n8n-nodes-base.code` | Data Standardization | When Security Alert Posted | Fetch AbuseIPDB Data | |
| Fetch AbuseIPDB Data | `n8n-nodes-base.httpRequest` | IP Reputation Lookup | Normalize Security Alert | Fetch IP Geo Information | |
| Fetch IP Geo Information | `n8n-nodes-base.httpRequest` | Geolocation & ASN Lookup | Fetch AbuseIPDB Data | Fetch VirusTotal File Data | |
| Fetch VirusTotal File Data | `n8n-nodes-base.httpRequest` | File Hash Analysis | Fetch IP Geo Information | Consolidate Data Enrichment | |
| Consolidate Data Enrichment | `n8n-nodes-base.code` | Data Aggregation | Fetch VirusTotal File Data | Security Triage Agent | |
| Security Triage Agent | `@n8n/n8n-nodes-langchain.agent` | AI Triage Orchestration | Consolidate Data Enrichment | Parse Triage Results | |
| OpenAI Security Model | `@n8n/n8n-nodes-langchain.lmChatOpenAi` | LLM Provider Integration | None (AI Link) | Security Triage Agent | |
| Parse Triage Results | `n8n-nodes-base.code` | JSON Parsing & Formatting | Security Triage Agent | Return Webhook Response, Check Severity Level | |
| Return Webhook Response | `n8n-nodes-base.respondToWebhook` | Webhook Responder | Parse Triage Results | None | |
| Check Severity Level | `n8n-nodes-base.if` | Conditional Severity Router | Parse Triage Results | Post to Slack Urgent Channel, Post to Slack Routine Log | |
| Post to Slack Urgent Channel | `n8n-nodes-base.httpRequest` | Urgent Slack Escalation | Check Severity Level | Send Incident Ticket | |
| Send Incident Ticket | `n8n-nodes-base.httpRequest` | Automated Incident Creation | Post to Slack Urgent Channel | None | |
| Post to Slack Routine Log | `n8n-nodes-base.httpRequest` | Routine Slack Logging | Check Severity Level | None | |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Trigger:**
   - Create a new node of type `Webhook` (`n8n-nodes-base.webhook`).
   - Name it `When Security Alert Posted`.
   - Set the HTTP Method to `POST` and path to `/security-alert`.
2. **Add Normalization Node:**
   - Create a `Code` node (`n8n-nodes-base.code`) named `Normalize Security Alert`.
   - Connect `When Security Alert Posted` to this node.
   - Insert JavaScript code to extract and map incoming fields (IDs, source IPs, file hashes, users, hostnames) into a standard schema.
3. **Configure AbuseIPDB Lookup:**
   - Create an `HTTP Request` node (`n8n-nodes-base.httpRequest`) named `Fetch AbuseIPDB Data`.
   - Connect `Normalize Security Alert` to this node.
   - Configure method as `GET`, URL pointing to the AbuseIPDB check endpoint, and add your AbuseIPDB API key header.
4. **Configure IP Geolocation Lookup:**
   - Create an `HTTP Request` node (`n8n-nodes-base.httpRequest`) named `Fetch IP Geo Information`.
   - Connect `Fetch AbuseIPDB Data` to this node.
   - Configure method as `GET`, pointing to the `ipinfo.io` endpoint for the normalized IP.
5. **Configure VirusTotal Lookup:**
   - Create an `HTTP Request` node (`n8n-nodes-base.httpRequest`) named `Fetch VirusTotal File Data`.
   - Connect `Fetch IP Geo Information` to this node.
   - Configure method as `GET`, pointing to the VirusTotal file report endpoint using the normalized file hash and required API key headers.
6. **Consolidate Enrichment Data:**
   - Create a `Code` node (`n8n-nodes-base.code`) named `Consolidate Data Enrichment`.
   - Connect `Fetch VirusTotal File Data` to this node.
   - Insert code to merge the outputs of the previous three API calls into a single structured alert object.
7. **Set Up AI Triage Agent:**
   - Create an `AI Agent` node (`@n8n/n8n-nodes-langchain.agent`) named `Security Triage Agent`.
   - Connect `Consolidate Data Enrichment` to this node's main input.
   - Configure the system prompt to instruct the model to perform SOC-style triage, returning a JSON structure containing severity, false-positive likelihood, summary, recommended actions, and confidence score.
8. **Configure OpenAI Language Model:**
   - Create an `OpenAI Chat Model` node (`@n8n/n8n-nodes-langchain.lmChatOpenAi`) named `OpenAI Security Model`.
   - Connect its AI language model output to the `Security Triage Agent`.
   - Configure the node with an OpenAI credential and select your desired chat model (e.g., `gpt-4o`).
9. **Parse AI Triage Results:**
   - Create a `Code` node (`n8n-nodes-base.code`) named `Parse Triage Results`.
   - Connect `Security Triage Agent` to this node.
   - Insert JavaScript code to parse the LLM text output into a clean JSON object.
10. **Configure Webhook Response:**
    - Create a `Respond to Webhook` node (`n8n-nodes-base.respondToWebhook`) named `Return Webhook Response`.
    - Connect `Parse Triage Results` to this node.
    - Set response body to output the enriched alert and triage JSON.
11. **Configure Severity Branching:**
    - Create an `If` node (`n8n-nodes-base.if`) named `Check Severity Level`.
    - Connect `Parse Triage Results` to this node.
    - Set condition rules to evaluate whether the triage severity is equal to `High` or `Critical`.
12. **Configure Urgent Escalation Path:**
    - Create an `HTTP Request` node named `Post to Slack Urgent Channel`.
    - Connect the True output of `Check Severity Level` to this node.
    - Configure method as `POST`, pointing to your urgent Slack incoming webhook URL with a formatted alert message.
    - Create a subsequent `HTTP Request` node named `Send Incident Ticket`.
    - Connect `Post to Slack Urgent Channel` to this node.
    - Configure method as `POST`, pointing to your internal ticketing system API endpoint with the incident payload.
13. **Configure Routine Logging Path:**
    - Create an `HTTP Request` node named `Post to Slack Routine Log`.
    - Connect the False output of `Check Severity Level` to this node.
    - Configure method as `POST`, pointing to your routine Slack incoming webhook URL for logging lower-severity events.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| External API Credentials Required | Ensure valid API keys and credentials are configured for AbuseIPDB, VirusTotal, ipinfo.io, OpenAI, and Slack webhooks before activating the workflow. |
| Ticketing Endpoint Customization | Adjust the JSON payload and URL in the `Send Incident Ticket` node to match your organization's specific ITSM/ticketing system requirements (e.g., Jira, ServiceNow, or custom APIs). |
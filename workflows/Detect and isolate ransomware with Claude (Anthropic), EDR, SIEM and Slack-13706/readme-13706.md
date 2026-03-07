Detect and isolate ransomware with Claude (Anthropic), EDR, SIEM and Slack

https://n8nworkflows.xyz/workflows/detect-and-isolate-ransomware-with-claude--anthropic---edr--siem-and-slack-13706


# Detect and isolate ransomware with Claude (Anthropic), EDR, SIEM and Slack

This technical document provides a comprehensive breakdown of the **AI Ransomware Early Warning System** workflow in n8n.

---

### 1. Workflow Overview
This workflow is designed for real-time detection and automated containment of ransomware attacks. It consumes file system event streams, aggregates behavioral data, and utilizes Claude AI (Anthropic) to perform high-fidelity threat analysis. Depending on a calculated threat score, the system either triggers automated network isolation and forensic capture via EDR/SIEM integrations or escalates the event for enhanced monitoring.

**Logical Blocks:**
*   **1.1 Data Ingestion & Aggregation:** Receives raw file events and calculates metrics (entropy, I/O velocity, extension changes).
*   **1.2 AI-Driven Threat Intelligence:** Leverages LLMs to evaluate behaviors against MITRE ATT&CK patterns.
*   **1.3 Decision Logic:** Routes the incident based on threat severity (threshold: 75/100).
*   **1.4 Automated Containment & Forensics:** Executes system isolation, terminates malicious processes, and captures system snapshots.
*   **1.5 Multi-Channel Incident Notification:** Alerts SOC teams via Slack, Email, PagerDuty, and logs data to SIEM and Google Sheets.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Ingestion & Metric Aggregation
*   **Overview:** This block acts as the entry point, receiving raw system logs and transforming them into a structured behavioral profile.
*   **Nodes Involved:** `File System Event Stream`, `Aggregate File Operations (30s Window)`, `Wait for Batch Window (30s)`.
*   **Node Details:**
    *   **File System Event Stream (Webhook):** Listens for POST requests at `/ransomware/file-events`. Expects JSON containing file operations (create, modify, rename, delete) and metadata (entropy, PID).
    *   **Aggregate File Operations (Code):** A complex JavaScript node that processes the event array. It identifies suspicious extensions (e.g., `.lockbit`, `.crypted`), detects ransom note patterns, and calculates I/O rates. It sets the `requires_ai_analysis` flag if thresholds are met.
    *   **Wait for Batch Window (Wait):** Pauses execution to ensure a full 30-second window of data is collected before AI analysis.

#### 2.2 AI-Powered Analysis
*   **Overview:** Interprets the aggregated metrics to distinguish between legitimate administrative activity and malicious encryption.
*   **Nodes Involved:** `Claude AI Ransomware Threat Analysis`, `Claude AI Model`, `Parse AI Threat Assessment`.
*   **Node Details:**
    *   **Claude AI Ransomware Threat Analysis (AI Agent):** A LangChain agent using a detailed system prompt. It evaluates the risk indicators, maps them to MITRE ATT&CK techniques (T1486, T1490), and estimates the ransomware family.
    *   **Claude AI Model (Anthropic):** Configured to use `claude-sonnet-4-20250514` with a low temperature (0.1) for deterministic security analysis.
    *   **Parse AI Threat Assessment (Code):** Cleans the AI output, extracts the JSON response, and generates a standardized `threatReport` object.

#### 2.3 Decision & Routing
*   **Overview:** Determines the response path based on the AI's confidence and threat score.
*   **Nodes Involved:** `Threat Score >= 75? (Auto-Isolate Threshold)`, `Confirm Isolation Required`.
*   **Node Details:**
    *   **Threat Score >= 75? (IF):** Evaluates the numerical score. Scores ≥ 75 trigger the high-severity path; lower scores trigger enhanced monitoring.
    *   **Confirm Isolation Required (IF):** A secondary check ensuring that either the boolean `isolation_required` is true or the recommendation is `ISOLATE_IMMEDIATELY`.

#### 2.4 Automated Containment (The "High Severity" Path)
*   **Overview:** Performs rapid response actions to stop data exfiltration and encryption.
*   **Nodes Involved:** `Capture Forensic Snapshot`, `Execute System Isolation`, `Terminate Encryption Process`.
*   **Node Details:**
    *   **Capture Forensic Snapshot (HTTP Request):** Calls the EDR API to preserve memory, process trees, and network connection logs before the system is isolated.
    *   **Execute System Isolation (HTTP Request):** Commands the EDR to perform "network_full" isolation, blocking adapters and SMB shares.
    *   **Terminate Encryption Process (HTTP Request):** Target-kills the specific PIDs identified by the AI as high-I/O encryption engines.

#### 2.5 Incident Logging & Notification
*   **Overview:** Ensures all stakeholders are notified and an audit trail is created.
*   **Nodes Involved:** `Alert SOC...`, `Email Security Team`, `Trigger PagerDuty...`, `Forward to SIEM`, `Write to Isolation Audit Log`, `Build Incident Response Summary`, `Send Detection Response`.
*   **Node Details:**
    *   **Multi-Channel Alerts:** Dispatches critical messages to Slack (v2.2), SMTP (v2.1), and PagerDuty (HTTP).
    *   **Forward to SIEM (HTTP):** Sends the full `threatReport` to a centralized SIEM (Splunk/Elastic).
    *   **Write to Isolation Audit Log (Google Sheets):** Appends incident data to a spreadsheet for compliance and post-incident review.
    *   **Send Detection Response (Respond to Webhook):** Closes the initial request with a full JSON summary of the actions taken.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| File System Event Stream | Webhook | Data Entry | (None) | Aggregate File Ops | 1. File System Monitoring & Event Collection |
| Aggregate File Operations | Code | Data Normalization | File System Event Stream | Wait | 1. File System Monitoring & Event Collection |
| Wait for Batch Window | Wait | Timing Control | Aggregate File Ops | Claude AI Agent | 1. File System Monitoring & Event Collection |
| Claude AI Agent | AI Agent | Threat Analysis | Wait | Parse AI Assessment | 2. Behavior Aggregation + AI Threat Analysis |
| Claude AI Model | Anthropic | AI Logic Engine | (None) | Claude AI Agent | 2. Behavior Aggregation + AI Threat Analysis |
| Parse AI Assessment | Code | Output Sanitization | Claude AI Agent | Threat Score IF | 2. Behavior Aggregation + AI Threat Analysis |
| Threat Score >= 75? | IF | High-Level Routing | Parse AI Assessment | Confirm Isolation / Monitoring | 3. Threat Scoring + Auto-Isolation Decision |
| Confirm Isolation | IF | Safety Check | Threat Score IF | Capture Forensic Snapshot | 3. Threat Scoring + Auto-Isolation Decision |
| Capture Forensic Snapshot | HTTP Request | Evidence Collection | Confirm Isolation | Execute Isolation | 4. System Isolation + Forensics + SOC Alert |
| Execute Isolation | HTTP Request | Containment | Capture Forensic Snapshot | Terminate Process | 4. System Isolation + Forensics + SOC Alert |
| Terminate Process | HTTP Request | Process Neutralization | Execute Isolation | Slack / Email / PagerDuty | 4. System Isolation + Forensics + SOC Alert |
| Alert SOC (Critical) | Slack | Real-time Alert | Terminate Process | Forward to SIEM | 4. System Isolation + Forensics + SOC Alert |
| Email Security Team | Email | Official Notification | Terminate Process | Forward to SIEM | 4. System Isolation + Forensics + SOC Alert |
| Trigger PagerDuty | HTTP Request | On-call Escalation | Terminate Process | Forward to SIEM | 4. System Isolation + Forensics + SOC Alert |
| Forward to SIEM | HTTP Request | Log Centralization | Slack / Email / PagerDuty | Write Audit Log | 4. System Isolation + Forensics + SOC Alert |
| Write Audit Log | Google Sheets | Compliance Logging | Forward to SIEM | Build Summary | 4. System Isolation + Forensics + SOC Alert |
| Build Summary | Code | Final Report Creation | Write Audit Log | Send Detection Response | 4. System Isolation + Forensics + SOC Alert |
| Send Detection Response | Respond to Webhook | Transaction Close | Build Summary | (None) | 4. System Isolation + Forensics + SOC Alert |
| Enhanced Monitoring | Code | Status Update | Threat Score IF | Notify SOC (Monitoring) | 3. Threat Scoring + Auto-Isolation Decision |
| Notify SOC (Monitoring) | Slack | Warning Alert | Enhanced Monitoring | Log Monitoring Alert | 3. Threat Scoring + Auto-Isolation Decision |
| Log Monitoring Alert | Google Sheets | Low-Level Logging | Notify SOC (Monitoring) | (None) | 3. Threat Scoring + Auto-Isolation Decision |

---

### 4. Reproducing the Workflow from Scratch

1.  **Ingestion Setup:** Create a **Webhook** node (Method: POST). Add a **Code** node following it to aggregate metrics. Use the JS logic to calculate file entropy and track PIDs.
2.  **Timing:** Add a **Wait** node set to 30 seconds to allow the batching of file events.
3.  **AI Integration:**
    *   Create an **AI Agent** node.
    *   Connect an **Anthropic Chat Model** node (Model: `claude-3-sonnet`).
    *   Define the System Prompt to require a strict JSON output covering `threat_score`, `mitre_attack_techniques`, and `recommended_action`.
4.  **Parsing:** Add a **Code** node to sanitize the AI's response (removing markdown backticks) and map it to a flat JSON object.
5.  **Routing:** Add an **IF** node checking `threat_score >= 75`.
6.  **Response Pipeline (True Branch):**
    *   Add **HTTP Request** nodes for EDR interaction (Snapshots, Isolation, Process Kill).
    *   Connect **Slack**, **Email**, and **PagerDuty** nodes in parallel for redundancy.
    *   Add a **Google Sheets** node for the audit log.
7.  **Response Pipeline (False Branch):**
    *   Add a **Code** node to set the status to `ENHANCED_MONITORING`.
    *   Add a **Slack** node for low-priority notification.
8.  **Completion:** End the workflow with a **Respond to Webhook** node to return the incident summary to the event source.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| MITRE ATT&CK Mapping | T1486 (Data Encrypted for Impact), T1490 (Inhibit System Recovery) |
| Detection Capabilities | Entropy Analysis (>7.5), I/O Velocity (>8 ops/sec), Shadow Copy Deletion detection |
| Forensic Standards | NIST CSF Alignment: DE.CM-7 (Monitoring), RS.MI-3 (Containment) |
| System Integration | Designed for EDRs like CrowdStrike, Defender, or SentinelOne via API |
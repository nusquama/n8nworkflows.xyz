Contract Extraction, Risk Analysis & Compliance Automation with GPT-4.1 Mini

https://n8nworkflows.xyz/workflows/contract-extraction--risk-analysis---compliance-automation-with-gpt-4-1-mini-10761


# Contract Extraction, Risk Analysis & Compliance Automation with GPT-4.1 Mini

---

### 1. Workflow Overview

This workflow automates contract extraction, risk analysis, and compliance auditing using GPT-4.1 Mini. It targets legal, compliance, and risk management teams needing to process contracts frequently and systematically. The workflow ingests contracts from multiple sources, extracts and chunks contract content, runs AI-based compliance checks, calculates risk scores, routes results for alerting and updates enterprise systems, and maintains audit trails for governance.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger & Configuration:** Initiates periodic contract review cycles and sets configuration parameters.
- **1.2 Contract Intake:** Collects contracts from repository API, webhook uploads, and Gmail contract-related emails.
- **1.3 Contract Content Extraction & Processing:** Extracts text from contract files, splits into chunks, generates embeddings, and stores vectors for semantic search.
- **1.4 AI Compliance Analysis:** Uses an AI agent powered by GPT-4.1 Mini to analyze contract clauses for compliance, missing clauses, deviations, and risks.
- **1.5 Risk Scoring:** Calculates a numerical risk score based on compliance findings and severity levels.
- **1.6 Result Routing & Notifications:** Routes contracts by risk level, sends Slack alerts for high-risk contracts, and posts data to dashboard APIs.
- **1.7 Enterprise System Updates & Audit Trail:** Updates CLM and ERP systems with compliance results, stores audit trails in a database for traceability.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Scheduled Trigger & Configuration

- **Overview:** This block triggers the workflow at a scheduled time and initializes configuration variables to be used throughout the workflow.
- **Nodes Involved:**  
  - Schedule Contract Review  
  - Workflow Configuration

- **Node Details:**

  - **Schedule Contract Review**  
    - Type: Schedule Trigger  
    - Role: Initiates workflow once daily at 2 AM to automate contract review.  
    - Configuration: Trigger set to hour 2 daily.  
    - Inputs: None (trigger node)  
    - Outputs: Connected to Workflow Configuration.  
    - Edge Cases: Workflow may not trigger if server time is misconfigured.  
    - Version: 1.2

  - **Workflow Configuration**  
    - Type: Set  
    - Role: Defines key API URLs, thresholds, and Slack channel ID as workflow-wide constants.  
    - Configuration: Sets placeholder strings for repository API, dashboard API, CLM system URL, ERP system URL, compliance rules source, risk thresholds (high=80, medium=50), and Slack channel ID for alerts.  
    - Inputs: From Schedule Contract Review  
    - Outputs: To Fetch Contracts from Repository  
    - Edge Cases: Missing or incorrect URLs will cause downstream HTTP requests to fail. Placeholder values must be replaced before production.  
    - Version: 3.4

---

#### 2.2 Contract Intake

- **Overview:** Gathers contracts from multiple sources — repository API, webhook uploads, and Gmail contract-related emails — and merges these into a unified dataset.
- **Nodes Involved:**  
  - Fetch Contracts from Repository  
  - Webhook - Contract Upload  
  - Gmail - Contract Emails  
  - Merge All Contract Sources

- **Node Details:**

  - **Fetch Contracts from Repository**  
    - Type: HTTP Request  
    - Role: Retrieves contracts stored in a repository via an API URL configured in Workflow Configuration.  
    - Configuration: Sends a GET request with JSON content-type header; expects file response format.  
    - Inputs: From Workflow Configuration  
    - Outputs: To Merge All Contract Sources  
    - Edge Cases: HTTP errors, invalid URLs, or network issues may cause failure. Authentication not configured here, so API must be publicly accessible or credentials added.  
    - Version: 4.3

  - **Webhook - Contract Upload**  
    - Type: Webhook  
    - Role: Receives contract files uploaded externally via POST requests to the "contract-upload" path.  
    - Configuration: HTTP POST method, response mode is last node output.  
    - Inputs: External HTTP requests  
    - Outputs: To Merge All Contract Sources  
    - Edge Cases: Invalid or malformed uploads, large file size limits, and security concerns (authentication not shown).  
    - Version: 2.1

  - **Gmail - Contract Emails**  
    - Type: Gmail Trigger  
    - Role: Polls Gmail every 5 minutes for emails with subject containing "contract" to ingest contract attachments.  
    - Configuration: Poll interval every 5 minutes, filter on subject "contract".  
    - Inputs: Autonomous polling  
    - Outputs: To Merge All Contract Sources  
    - Credentials: Gmail OAuth2 configured  
    - Edge Cases: Gmail API rate limits, authentication expiry, missing attachments.  
    - Version: 1.3

  - **Merge All Contract Sources**  
    - Type: Merge  
    - Role: Combines the three sources of contract data into a single stream for downstream processing.  
    - Configuration: Accepts 3 inputs, merges them into one output.  
    - Inputs: From Fetch Contracts, Webhook, Gmail nodes  
    - Outputs: To Extract Contract Content  
    - Edge Cases: Input data format mismatches or empty inputs could cause downstream errors.  
    - Version: 3.2

---

#### 2.3 Contract Content Extraction & Processing

- **Overview:** Extracts textual content from contract files (mainly PDFs), splits text into manageable chunks, generates semantic embeddings for each chunk, and stores them in an in-memory vector store for AI querying.
- **Nodes Involved:**  
  - Extract Contract Content  
  - Vector Store - Contract Clauses  
  - Text Splitter - Clause Chunking  
  - Document Loader - Contract Processing  
  - OpenAI Embeddings

- **Node Details:**

  - **Extract Contract Content**  
    - Type: Extract From File  
    - Role: Converts PDF or image contract files into machine-readable text.  
    - Configuration: Operation set to PDF extraction.  
    - Inputs: From Merge All Contract Sources  
    - Outputs: To Vector Store - Contract Clauses  
    - Edge Cases: OCR failures for poor-quality scans, unsupported file types, large files causing timeouts.  
    - Version: 1

  - **Vector Store - Contract Clauses**  
    - Type: Vector Store In-Memory (Langchain)  
    - Role: Stores contract clause embeddings for semantic search and AI analysis.  
    - Configuration: Mode insert, memory key "vector_store_key".  
    - Inputs: From Extract Contract Content and Document Loader  
    - Outputs: To AI Agent - Compliance Checker  
    - Edge Cases: High memory consumption for large contracts, potential data loss on workflow restart (in-memory).  
    - Version: 1.3

  - **Text Splitter - Clause Chunking**  
    - Type: Recursive Character Text Splitter (Langchain)  
    - Role: Splits contract text into 2000-character chunks with 200-character overlap, preserving context.  
    - Inputs: AI document input from Document Loader  
    - Outputs: AI document output to Document Loader  
    - Edge Cases: Incorrect chunk overlap could split clauses awkwardly, impacting AI comprehension.  
    - Version: 1

  - **Document Loader - Contract Processing**  
    - Type: Document Default Data Loader (Langchain)  
    - Role: Loads binary contract data, applies custom text splitting mode.  
    - Inputs: From Text Splitter  
    - Outputs: To Vector Store  
    - Edge Cases: Binary data format errors, large files causing slow loading.  
    - Version: 1.1

  - **OpenAI Embeddings**  
    - Type: OpenAI Embeddings (Langchain)  
    - Role: Generates vector embeddings of contract text for semantic understanding.  
    - Inputs: To Vector Store (embedding input)  
    - Credentials: OpenAI API key required  
    - Edge Cases: API rate limits, authentication failures, large text causing truncation.  
    - Version: 1.2

---

#### 2.4 AI Compliance Analysis

- **Overview:** Runs a GPT-4.1 Mini powered AI agent to analyze contract clauses for compliance status, missing clauses, deviations from best practices, legal risks, and actionable recommendations. Outputs structured JSON results.
- **Nodes Involved:**  
  - AI Agent - Compliance Checker  
  - OpenAI Chat Model - Compliance Analysis  
  - Structured Parser - Compliance Results

- **Node Details:**

  - **AI Agent - Compliance Checker**  
    - Type: Langchain Agent  
    - Role: Orchestrates AI analysis using a defined prompt to identify contract compliance issues based on extracted text.  
    - Configuration: Custom prompt instructs the agent to identify key clauses, check compliance, missing clauses, deviations, flag risks, assess severity, and recommend actions. Uses extracted text as input.  
    - Inputs: From Vector Store - Contract Clauses (semantic search results)  
    - Outputs: To Calculate Risk Score  
    - Has structured output parser enabled.  
    - Credentials: OpenAI API key required  
    - Edge Cases: AI hallucination, incomplete text input, prompt misinterpretation, API errors.  
    - Version: 3

  - **OpenAI Chat Model - Compliance Analysis**  
    - Type: Langchain OpenAI Chat Model  
    - Role: Provides the GPT-4.1 Mini language model backend for the AI Agent.  
    - Inputs: AI language model input from AI Agent  
    - Outputs: AI Agent for processing  
    - Credentials: OpenAI API key required  
    - Edge Cases: API rate limits, model availability, network timeouts.  
    - Version: 1.2

  - **Structured Parser - Compliance Results**  
    - Type: Structured Output Parser (Langchain)  
    - Role: Parses AI JSON output into a structured schema with contractId, complianceStatus, findings, missingClauses, deviations, summary.  
    - Inputs: AI output parser input from AI Agent  
    - Outputs: AI Agent continuation  
    - Edge Cases: Parsing errors if AI output format deviates from schema, JSON syntax issues.  
    - Version: 1.3

---

#### 2.5 Risk Scoring

- **Overview:** Calculates a quantitative risk score (0–100) and risk level (CRITICAL, HIGH, MEDIUM, LOW) based on AI findings. Determines business impact and review priority for each contract.
- **Nodes Involved:**  
  - Calculate Risk Score

- **Node Details:**

  - **Calculate Risk Score**  
    - Type: Code (JavaScript)  
    - Role: Implements a weighted scoring algorithm based on compliance status, number and severity of findings, missing clauses, and deviations.  
    - Configuration:  
      - Assigns points for non-compliance, partial compliance, findings severity (high=30, medium=15, low=5), missing clauses, deviations.  
      - Caps score at 100.  
      - Determines risk level and business impact.  
      - Flags contracts requiring urgent legal review.  
    - Inputs: From AI Agent - Compliance Checker  
    - Outputs: To Store Compliance Results  
    - Edge Cases: Missing or malformed input data, code errors, empty findings array.  
    - Version: 2

---

#### 2.6 Result Routing & Notifications

- **Overview:** Routes contracts based on risk level for appropriate handling, sends Slack alerts for high-risk contracts, and posts compliance data to a dashboard API.
- **Nodes Involved:**  
  - Store Compliance Results  
  - Route by Risk Level  
  - Alert - High Risk Contracts  
  - Send to Dashboard API

- **Node Details:**

  - **Store Compliance Results**  
    - Type: Postgres  
    - Role: Persists compliance scoring results into a database table.  
    - Configuration: Table and schema placeholders must be set (e.g., Compliance Results Table).  
    - Inputs: From Calculate Risk Score  
    - Outputs: To Route by Risk Level  
    - Edge Cases: DB connection failures, schema mismatches, data mapping errors.  
    - Version: 2.6

  - **Route by Risk Level**  
    - Type: Switch  
    - Role: Routes contract records based on riskLevel field (CRITICAL, HIGH, MEDIUM, LOW).  
    - Configuration: Conditions test equality of riskLevel; fallback route is Low Risk.  
    - Inputs: From Store Compliance Results  
    - Outputs: To Alert - High Risk Contracts and Send to Dashboard API (for high and critical), others follow fallback.  
    - Edge Cases: Unexpected riskLevel values, missing fields.  
    - Version: 3.3

  - **Alert - High Risk Contracts**  
    - Type: Slack  
    - Role: Sends formatted Slack alert messages to a configured channel for critical and high-risk contracts.  
    - Configuration: Uses Slack OAuth2 credentials, sends message with contract details, risk factors, and action required.  
    - Inputs: From Route by Risk Level (Critical/High Risk output)  
    - Outputs: None  
    - Edge Cases: Slack API rate limits, invalid channel ID, authentication failure.  
    - Version: 2.3

  - **Send to Dashboard API**  
    - Type: HTTP Request  
    - Role: Sends compliance results JSON to a dashboard API endpoint configured in workflow settings.  
    - Configuration: POST with JSON body, Content-Type header set.  
    - Inputs: From Route by Risk Level (Critical/High Risk output)  
    - Outputs: To Update CLM System  
    - Edge Cases: HTTP errors, invalid URL, API authentication missing.  
    - Version: 4.3

---

#### 2.7 Enterprise System Updates & Audit Trail

- **Overview:** Updates Contract Lifecycle Management (CLM) and ERP systems with compliance and risk data; prepares and stores audit trail records in a database for governance.
- **Nodes Involved:**  
  - Update CLM System  
  - Update ERP System  
  - Prepare Audit Trail  
  - Store Audit Trail

- **Node Details:**

  - **Update CLM System**  
    - Type: HTTP Request  
    - Role: Sends PUT request to CLM system API to update contract compliance status, risk score, and last assessment timestamp.  
    - Inputs: From Send to Dashboard API  
    - Outputs: To Update ERP System  
    - Configuration: JSON body constructed with contractId, complianceStatus, riskScore, assessment timestamp.  
    - Edge Cases: API errors, invalid payload, authentication issues.  
    - Version: 4.3

  - **Update ERP System**  
    - Type: HTTP Request  
    - Role: Sends POST request to ERP system to update risk level and flags if legal review is required.  
    - Inputs: From Update CLM System  
    - Outputs: To Prepare Audit Trail  
    - Configuration: JSON body with contractId, riskLevel, requiresAction boolean.  
    - Edge Cases: ERP API errors, payload formatting errors.  
    - Version: 4.3

  - **Prepare Audit Trail**  
    - Type: Set  
    - Role: Constructs audit trail metadata including auditId (timestamp + contractId), workflow execution ID, processor info, timestamp, retention period (2555 days), and explainability report as JSON string.  
    - Inputs: From Update ERP System  
    - Outputs: To Store Audit Trail  
    - Edge Cases: Missing input fields, JSON serialization errors.  
    - Version: 3.4

  - **Store Audit Trail**  
    - Type: Postgres  
    - Role: Persists audit trail records into a database table for compliance tracking and governance.  
    - Inputs: From Prepare Audit Trail  
    - Configuration: Table and schema placeholders must be set (Audit Trail Table).  
    - Edge Cases: Database connection failures, data mapping errors.  
    - Version: 2.6

---

### 3. Summary Table

| Node Name                     | Node Type                              | Functional Role                                       | Input Node(s)                        | Output Node(s)                         | Sticky Note                                   |
|-------------------------------|--------------------------------------|------------------------------------------------------|------------------------------------|---------------------------------------|-----------------------------------------------|
| Schedule Contract Review       | Schedule Trigger                     | Initiates scheduled workflow execution                | None                               | Workflow Configuration                 | Schedule Contract Review Cycle: triggers workflow daily to batch-process contracts |
| Workflow Configuration        | Set                                 | Sets API URLs, thresholds, and Slack channel          | Schedule Contract Review            | Fetch Contracts from Repository        |                                               |
| Fetch Contracts from Repository| HTTP Request                        | Fetches contracts from API repository                  | Workflow Configuration             | Merge All Contract Sources              | Fetch & Ingest Contracts: centralized contract intake from multiple sources |
| Webhook - Contract Upload      | Webhook                             | Receives contract uploads via HTTP POST               | External HTTP request              | Merge All Contract Sources              |                                               |
| Gmail - Contract Emails        | Gmail Trigger                       | Polls Gmail for contract-related emails                | Autonomous polling                 | Merge All Contract Sources              |                                               |
| Merge All Contract Sources     | Merge                              | Merges contracts from repository, webhook, and email  | Fetch Contracts, Webhook, Gmail    | Extract Contract Content                |                                               |
| Extract Contract Content       | Extract From File                   | Extracts text from contract PDFs                        | Merge All Contract Sources         | Vector Store - Contract Clauses         | OCR + AI Content Extraction: converts contracts to machine-readable text |
| Vector Store - Contract Clauses| Vector Store In-Memory (Langchain) | Stores embeddings for contract clauses                 | Extract Contract Content, Document Loader | AI Agent - Compliance Checker       | Parse Obligations & Covenants: semantic search storage for legal clauses |
| Text Splitter - Clause Chunking| Recursive Character Text Splitter   | Splits contract text into overlapping chunks           | Document Loader                   | Document Loader                        |                                               |
| Document Loader - Contract Processing | Document Default Data Loader (Langchain) | Loads binary contract data and applies text splitting | Text Splitter                    | Vector Store - Contract Clauses         |                                               |
| OpenAI Embeddings              | OpenAI Embeddings (Langchain)       | Generates semantic embeddings of contract text         | To Vector Store (embedding input) | Vector Store - Contract Clauses         |                                               |
| AI Agent - Compliance Checker  | Langchain Agent                    | Runs compliance analysis AI agent on contract clauses  | Vector Store - Contract Clauses   | Calculate Risk Score                   | Multi-Model Compliance Check: runs AI models for GDPR, SOX, industry rules |
| OpenAI Chat Model - Compliance Analysis | Langchain OpenAI Chat Model    | GPT-4.1 Mini language model backend                     | AI Agent (language model input)    | AI Agent                              |                                               |
| Structured Parser - Compliance Results | Structured Output Parser (Langchain) | Parses AI JSON output into structured compliance data | AI Agent (output parser input)     | AI Agent                              |                                               |
| Calculate Risk Score           | Code (JavaScript)                   | Computes numeric risk score and risk level              | AI Agent - Compliance Checker      | Store Compliance Results               | Calculate Risk Score: scores severity and prioritizes risk |
| Store Compliance Results       | Postgres                           | Stores compliance results in database                    | Calculate Risk Score               | Route by Risk Level                    |                                               |
| Route by Risk Level            | Switch                            | Routes contracts based on risk level                     | Store Compliance Results           | Alert - High Risk Contracts, Send to Dashboard API | Contract Risk Routing & Archiving: routes by risk and archives results |
| Alert - High Risk Contracts    | Slack                             | Sends Slack alerts for high-risk contracts               | Route by Risk Level                | None                                 | Alerts: sends risk alerts to leadership via Slack |
| Send to Dashboard API          | HTTP Request                      | Posts compliance data to dashboard API                   | Route by Risk Level                | Update CLM System                     | CLM Integration & Compliance Tracking: updates systems and logs data |
| Update CLM System              | HTTP Request                      | Updates contract status and risk score in CLM system    | Send to Dashboard API              | Update ERP System                    |                                               |
| Update ERP System              | HTTP Request                      | Updates ERP system with risk level and action flags     | Update CLM System                 | Prepare Audit Trail                   |                                               |
| Prepare Audit Trail            | Set                               | Prepares metadata for audit trail record                 | Update ERP System                 | Store Audit Trail                    |                                               |
| Store Audit Trail              | Postgres                         | Stores audit trail in database                            | Prepare Audit Trail               | None                                 |                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Schedule Contract Review" node**  
   - Type: Schedule Trigger  
   - Set trigger interval to daily at 2 AM.  
   - Connect output to "Workflow Configuration".

2. **Create "Workflow Configuration" node**  
   - Type: Set  
   - Assign variables:  
     - repositoryApiUrl: set to your contract repository API URL  
     - dashboardApiUrl: dashboard API endpoint  
     - clmSystemUrl: CLM system API URL  
     - erpSystemUrl: ERP system API URL  
     - complianceRulesUrl: compliance rules API or file URL  
     - riskThresholdHigh: 80  
     - riskThresholdMedium: 50  
     - slackChannel: Slack channel ID for alerts  
   - Connect output to "Fetch Contracts from Repository".

3. **Create "Fetch Contracts from Repository" node**  
   - Type: HTTP Request  
   - Method: GET (default)  
   - URL: reference `{{ $('Workflow Configuration').first().json.repositoryApiUrl }}`  
   - Headers: Content-Type: application/json  
   - Response format: file  
   - Connect output to "Merge All Contract Sources".

4. **Create "Webhook - Contract Upload" node**  
   - Type: Webhook  
   - Path: contract-upload  
   - HTTP Method: POST  
   - Response Mode: last node  
   - Connect output to "Merge All Contract Sources".

5. **Create "Gmail - Contract Emails" node**  
   - Type: Gmail Trigger  
   - Filter: subject contains "contract"  
   - Poll every 5 minutes  
   - Configure Gmail OAuth2 credentials  
   - Connect output to "Merge All Contract Sources".

6. **Create "Merge All Contract Sources" node**  
   - Type: Merge  
   - Number of inputs: 3 (from repository, webhook, Gmail)  
   - Connect output to "Extract Contract Content".

7. **Create "Extract Contract Content" node**  
   - Type: Extract From File  
   - Operation: pdf  
   - Connect output to "Vector Store - Contract Clauses".

8. **Create "Text Splitter - Clause Chunking" node**  
   - Type: Recursive Character Text Splitter (Langchain)  
   - Chunk size: 2000 characters  
   - Chunk overlap: 200 characters  
   - Connect AI textSplitter output to "Document Loader - Contract Processing".

9. **Create "Document Loader - Contract Processing" node**  
   - Type: Document Default Data Loader (Langchain)  
   - Data type: binary  
   - Text splitting mode: custom  
   - Connect AI document output to "Vector Store - Contract Clauses".

10. **Create "OpenAI Embeddings" node**  
    - Type: OpenAI Embeddings (Langchain)  
    - Connect output embedding to Vector Store (embedding input)  
    - Configure OpenAI API credentials.

11. **Create "Vector Store - Contract Clauses" node**  
    - Type: Vector Store In-Memory (Langchain)  
    - Mode: insert  
    - Memory key: vector_store_key  
    - Connect main output to "AI Agent - Compliance Checker".

12. **Create "AI Agent - Compliance Checker" node**  
    - Type: Langchain Agent  
    - Prompt: Define detailed compliance checking instructions including clause identification, compliance status, missing clauses, deviations, severity, recommendations, using extracted text.  
    - Enable structured output parser with JSON schema matching compliance results.  
    - Connect main output to "Calculate Risk Score".  
    - Configure OpenAI API credentials.

13. **Create "OpenAI Chat Model - Compliance Analysis" node**  
    - Type: Langchain OpenAI Chat Model  
    - Model: gpt-4.1-mini  
    - Connect ai_languageModel input/output to AI Agent node.  
    - Configure OpenAI API credentials.

14. **Create "Structured Parser - Compliance Results" node**  
    - Type: Structured Output Parser (Langchain)  
    - Provide JSON schema example for compliance results (contractId, complianceStatus, findings, missingClauses, deviations, summary).  
    - Connect ai_outputParser input/output to AI Agent node.

15. **Create "Calculate Risk Score" node**  
    - Type: Code (JavaScript)  
    - Paste the provided risk scoring algorithm code.  
    - Connect output to "Store Compliance Results".

16. **Create "Store Compliance Results" node**  
    - Type: Postgres  
    - Configure with compliance results table and schema.  
    - Connect output to "Route by Risk Level".

17. **Create "Route by Risk Level" node**  
    - Type: Switch  
    - Add conditions for riskLevel: CRITICAL, HIGH, MEDIUM, default LOW.  
    - Connect outputs:  
      - Critical and High Risk → Alert - High Risk Contracts and Send to Dashboard API  
      - Medium Risk and Low Risk → downstream as needed (Low risk fallback).

18. **Create "Alert - High Risk Contracts" node**  
    - Type: Slack  
    - Configure Slack OAuth2 credentials.  
    - Set message template with contract details and risk factors.  
    - Connect input from Route by Risk Level.

19. **Create "Send to Dashboard API" node**  
    - Type: HTTP Request  
    - Method: POST  
    - URL from Workflow Configuration dashboardApiUrl  
    - Content-Type: application/json  
    - Connect output to "Update CLM System".

20. **Create "Update CLM System" node**  
    - Type: HTTP Request  
    - Method: PUT  
    - URL from Workflow Configuration clmSystemUrl  
    - JSON body with contractId, complianceStatus, riskScore, lastAssessment timestamp  
    - Connect output to "Update ERP System".

21. **Create "Update ERP System" node**  
    - Type: HTTP Request  
    - Method: POST  
    - URL from Workflow Configuration erpSystemUrl  
    - JSON body with contractId, riskLevel, requiresAction boolean  
    - Connect output to "Prepare Audit Trail".

22. **Create "Prepare Audit Trail" node**  
    - Type: Set  
    - Assign: auditId (ISO timestamp + contractId), workflowExecutionId, processedBy ("AI Compliance Checker"), auditTimestamp (ISO now), dataRetentionDays (2555), explainabilityReport (JSON string of findings)  
    - Connect output to "Store Audit Trail".

23. **Create "Store Audit Trail" node**  
    - Type: Postgres  
    - Configure with audit trail table and schema.  
    - Connect input from Prepare Audit Trail.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                      | Context or Link                                                                                              |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| Scheduled triggers initiate automated contract reviews, fetching from cloud storage and email inboxes, then AI extracts terms and compliance requirements. Multi-model parsing identifies gaps and risks, scoring severity and routing alerts. Updates CLM and produces audit trails. | Overview sticky note at position [576,-416].                                                                |
| Setup instructions require configuring cloud storage access, Gmail credentials, OpenAI API keys, CLM system credentials, and dashboard endpoints.                                                                               | Setup Instructions sticky note at position [1168,-416].                                                     |
| Prerequisites include cloud storage, Gmail OAuth credentials, OpenAI API key, CLM system credentials, and document processing licenses. Use cases include contract renewal tracking, compliance audits, risk management, and vendor agreement reviews.                               | Prerequisites sticky note at position [1568,-416].                                                          |
| Customize risk thresholds, extraction rules, Slack notifications, and compliance frameworks to fit organizational needs. Workflow benefits include 80% reduction in review time and automated audit trails.                         | Customization sticky note at position [2016,-416].                                                          |
| Contract review is scheduled regularly to ensure no contracts remain unreviewed, supporting consistent compliance monitoring.                                                                                                   | Schedule Contract Review Cycle sticky note at position [384,-80].                                           |
| Contract ingestion centralizes documents from cloud storage, email, and direct uploads to ensure all contracts are analyzed, preventing missed files.                                                                           | Fetch & Ingest Contracts sticky note at position [784,-80].                                                 |
| OCR and AI extract structured data (parties, dates, terms, limits) from unstructured contract documents, enabling queryable data for analysis.                                                                                 | OCR + AI Content Extraction sticky note at position [1232,-80].                                            |
| Parsing obligations identifies key legal clauses such as payment schedules, performance metrics, termination conditions to prevent costly SLA breaches.                                                                        | Parse Obligations & Covenants sticky note at position [1744,-80].                                           |
| Multi-model compliance checks run against GDPR, SOX, and industry rules in parallel to detect gaps that single models may miss.                                                                                               | Multi-Model Compliance Check sticky note at position [2256,-80].                                           |
| Risk scoring assigns numeric ratings based on compliance gaps and financial exposure to prioritize contract reviews.                                                                                                           | Calculate Risk Score sticky note at position [2064,-80].                                                    |
| Contracts are routed by risk level, with high-risk contracts sent to legal, medium risk to compliance officers, and low risk archived for audits, ensuring appropriate handling.                                                 | Contract Risk Routing & Archiving sticky note at position [2416,-80].                                      |
| CLM system integration centralizes compliance data, risk scores, and review dates, improving data governance and AI model improvements over time.                                                                              | CLM Integration & Compliance Tracking sticky note at position [2704,48].                                   |
| Alerts summarize contract risk, gaps, and deadlines, enabling leadership to quickly focus on urgent issues.                                                                                                                    | Alerts sticky note at position [3200,48].                                                                   |

---

**Disclaimer:** This document is generated from an automated n8n workflow export. It complies with all content policies and does not contain any illegal or protected content. All data processed is legal and publicly accessible.

---
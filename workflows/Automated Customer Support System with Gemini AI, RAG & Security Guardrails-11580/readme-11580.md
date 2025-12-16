Automated Customer Support System with Gemini AI, RAG & Security Guardrails

https://n8nworkflows.xyz/workflows/automated-customer-support-system-with-gemini-ai--rag---security-guardrails-11580


# Automated Customer Support System with Gemini AI, RAG & Security Guardrails

### 1. Workflow Overview

This workflow is an advanced **Automated Customer Support System** leveraging Google Gemini AI models, Retrieval-Augmented Generation (RAG), and strict Security Guardrails to orchestrate multi-agentic handling of incoming customer email queries. Its primary purpose is to efficiently analyze, triage, escalate, and respond to customer support tickets while ensuring compliance, safety, and quality control.

**Target Use Cases:**  
- Automated processing of customer support emails  
- Detection and escalation of critical or hostile messages  
- Retrieval of company policy/facts for accurate responses  
- Drafting personalized, empathetic email replies  
- Enforcing security guardrails and logging sensitive content  

**Logical Blocks:**  
- **1.1 Input Reception & Security Guardrails:** Ingest emails via Gmail trigger and scan for PII, profanity, or prompt injection attempts. Log threats and halt unsafe requests.  
- **1.2 Master Orchestrator Agent:** The central brain controlling the flow across phases: triage, evaluation & routing, knowledge retrieval, synthesis and review, and final email sending.  
- **1.3 Ticket Triage Analyser Agent:** Analyzes customer messages for category, sentiment, urgency, and priority to inform routing decisions.  
- **1.4 Knowledge Worker & Investigation Agent:** Performs RAG-based knowledge retrieval from company policy documents stored in a vector DB.  
- **1.5 Resolution Agent:** Drafts polite, professional email responses based on retrieved facts and original customer input.  
- **1.6 Escalation & Notification Tools:** Slack alerts for critical cases, email reply tool for communication, and Airtable logging of flagged incidents.  
- **1.7 Knowledge Base Loader & Vector Storage:** Manual ingestion of corporate policy documents into vector DB for RAG operations.  
- **1.8 Supporting Utilities:** Memory buffers for session context, JSON cleaning scripts, and placeholders for sub-workflow inputs.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Security Guardrails

- **Overview:**  
  This block triggers on incoming emails, applies guardrails to detect sensitive data, prompt injections, or profanity, and either proceeds or logs threats and halts the workflow.

- **Nodes Involved:**  
  - Gmail Trigger  
  - Guardrails (LangChain Guardrails node)  
  - Log Threats in Airtable  
  - Guard LLM (Google Gemini LLM for guardrails)

- **Node Details:**  
  - **Gmail Trigger:**  
    - Type: Email trigger node  
    - Configured to poll every minute, filtering unread emails from a specific sender/email address  
    - Outputs email snippet and metadata for processing  
    - Inputs to Guardrails node  

  - **Guardrails:**  
    - Type: LangChain Guardrails node integrating AI safety checks  
    - Configured with custom guardrails for PII detection, prompt injection, and profanity/hate speech check  
    - Uses text from incoming email snippet  
    - Outputs either 'pass' or 'fail' results for each guardrail check  
    - If fail, branches to Log Threats in Airtable and halts workflow  

  - **Log Threats in Airtable:**  
    - Type: Airtable node (upsert operation)  
    - Logs incident type (PII, Profanity, Prompt Injection) along with email ID and sender info  
    - Requires Airtable personal access token credentials  
    - Failure cases: API errors, credential misconfigurations  

  - **Guard LLM:**  
    - Type: Google Gemini LLM node  
    - Supports guardrails with hate speech category setting  
    - Used internally by Guardrails for content safety evaluation  

- **Edge Cases & Failure Types:**  
  - Emails containing PII or prompt injection halt further processing  
  - Airtable API rate limits or token errors may cause logging failures  
  - Guardrails misclassification could cause false positives/negatives

---

#### 2.2 Master Orchestrator Agent

- **Overview:**  
  The central coordinating agent that maintains conversation state and enforces a strict 5-phase customer support process: triage, evaluation & routing, knowledge retrieval, response synthesis, and final email sending.

- **Nodes Involved:**  
  - Orchestrator Agent (LangChain Agent node)  
  - Slack Tool (Slack notification for escalations)  
  - Email Reply Tool (for sending emails)  
  - Simple Memory (session context for orchestrator)

- **Node Details:**  
  - **Orchestrator Agent:**  
    - Type: LangChain Agent  
    - Input: Original email snippet  
    - Uses a detailed system message defining the 5-phase sequence and enforcing strict rules on calling sub-agents and tools  
    - Calls sub-workflows for Ticket Triage, Knowledge Worker, Resolution Agent, Slack alerts, and Email replies as per logic  
    - Holds session memory keyed by email ID  
    - Version: LangChain Agent v3, supports multiple ai_tool calls  
    - Failure: AI reasoning errors, unexpected output, or tool call failures  

  - **Slack Tool:**  
    - Type: Slack API integration via n8n node  
    - Sends urgent alerts to a specific channel with details about critical escalation  
    - Input built by Orchestrator Agent as alertMessage string  
    - Requires Slack API OAuth credentials  
    - Failures: Slack API downtime, invalid channel ID  

  - **Email Reply Tool:**  
    - Type: Gmail Tool node (reply operation)  
    - Sends the final email response (either apology or drafted reply) to customer  
    - Uses messageId from incoming email for threading  
    - Requires Gmail OAuth2 credentials  
    - Failures: Gmail quota limits, auth errors  

  - **Simple Memory:**  
    - Type: LangChain memory buffer window  
    - Maintains conversation state for orchestrator agent using email ID as session key  

- **Edge Cases & Failures:**  
  - Orchestrator may fail if sub-agents return malformed data  
  - Slack or Gmail API failures cause notification or email sending issues  
  - Memory overflow or key collision errors

---

#### 2.3 Ticket Triage Analyser Agent

- **Overview:**  
  Specialized agent analyzing customer messages to classify category, urgency, sentiment, priority score, and summary, which drives routing decisions.

- **Nodes Involved:**  
  - Call Ticket Analyser Agent (toolWorkflow node)  
  - Email / Ticket Analyser Agent (LangChain Agent node)  
  - Ticket Analyser LLM (Google Gemini LLM node)  
  - Clean JSON (code node)  
  - Simple Memory for Email Analyser Agent  
  - Placeholder 1 (input placeholder set node)

- **Node Details:**  
  - **Call Ticket Analyser Agent:**  
    - Type: Tool Workflow node calling separate workflow by ID  
    - Inputs: Customer query snippet, threadId, senderEmail from Gmail Trigger  
    - Calls sub-workflow running Email / Ticket Analyser Agent  

  - **Email / Ticket Analyser Agent:**  
    - Type: LangChain Agent  
    - System message enforces output JSON schema with category, urgency, sentiment, priority_score, summary  
    - Uses Google Gemini LLM (via Ticket Analyser LLM node)  
    - Version 3 agent node  

  - **Clean JSON:**  
    - Type: Code node (JavaScript)  
    - Cleans raw AI agent output by stripping markdown code blocks and parsing JSON  
    - Handles parse errors by assigning default safe values (General Inquiry, Medium urgency, Neutral sentiment)  
    - Ensures downstream nodes receive structured data  

  - **Simple Memory for Email Analyser Agent:**  
    - Stores session context keyed by senderEmail + threadId  

  - **Placeholder 1:**  
    - Set node used as input schema placeholder in sub-workflow  

- **Edge Cases & Failures:**  
  - AI agent output may produce malformed JSON, handled by Clean JSON fallback  
  - Misclassification may affect downstream routing  
  - Sub-workflow invocation errors or missing workflow ID  

---

#### 2.4 Knowledge Worker & Investigation Agent (RAG)

- **Overview:**  
  Retrieves relevant facts and policy details from the company knowledge base using RAG, based on triage summary and customer query for non-escalated tickets.

- **Nodes Involved:**  
  - Call Knowledge Worker Agent (toolWorkflow node)  
  - Knowledge worker and Investigator (LangChain Agent node)  
  - RAG LLM (Google Gemini LLM node)  
  - Knowledge Base Retrieval (Supabase vector DB node)  
  - Clean Json (code node for cleaning RAG output)  
  - Simple Memory for RAG agent  
  - Placeholder 2 (input placeholder set node)

- **Node Details:**  
  - **Call Knowledge Worker Agent:**  
    - Calls separate tool workflow with inputs: query, senderEmail, threadId, triageSummary, triageCategory  
    - Only called for non-urgent, non-escalated cases  

  - **Knowledge worker and Investigator:**  
    - LangChain Agent defined as a retrieval machine  
    - Must use Knowledge Base Retrieval tool exclusively  
    - Outputs raw facts or "NO_DATA_FOUND" if empty  
    - Uses RAG LLM (Gemini) for retrieval augmented generation  

  - **Knowledge Base Retrieval:**  
    - Supabase vectorStore node operating in "retrieve-as-tool" mode  
    - Searches "n8n_documents" table for policy documents  
    - The only source of truth for facts and policy  

  - **Clean Json:**  
    - JavaScript code node that sanitizes LLM output by removing boilerplate phrases and isolating factual content  
    - Adds field "retrievedFacts" for downstream resolution agent  

  - **Simple Memory for RAG agent:**  
    - Maintains session context keyed by senderEmail + threadId  

  - **Placeholder 2:**  
    - Input placeholder node for sub-workflow  

- **Edge Cases & Failures:**  
  - Empty or no relevant data found leads to "NO_DATA_FOUND" output  
  - Vector DB connection or query failures  
  - LLM hallucination or irrelevant retrieval  

---

#### 2.5 Resolution Agent

- **Overview:**  
  Drafts the final polite, empathetic email response using the synthesized facts from the RAG agent combined with original customer query and triage summary.

- **Nodes Involved:**  
  - Call Resolution Agent (toolWorkflow node)  
  - Resolution Agent (LangChain Agent node)  
  - Resolution Agent LLM (Google Gemini LLM node)  
  - Simple Memory for Resolution Agent  
  - Placeholder 3 (input placeholder set node)

- **Node Details:**  
  - **Call Resolution Agent:**  
    - Calls separate workflow with inputs: ragFacts, customerQuery, triageSummary  
    - Only used for standard, non-escalated queries  

  - **Resolution Agent:**  
    - LangChain Agent tasked with drafting email body only (no sending)  
    - Strict constraints: no JSON output, no subject line, empathetic tone, double newlines for paragraphs  
    - Uses Gemini LLM (Resolution Agent LLM node)  

  - **Simple Memory for Resolution Agent:**  
    - Stores session context keyed by senderEmail + threadId  

  - **Placeholder 3:**  
    - Input placeholder node for sub-workflow  

- **Edge Cases & Failures:**  
  - AI writing errors or tone inconsistencies  
  - Possible contradiction between customer query and facts; agent instructed to trust RAG facts  
  - Sub-workflow call failures  

---

#### 2.6 Escalation & Notification Tools

- **Overview:**  
  Handles emergency escalation by notifying human managers via Slack and sending apology emails to customers.

- **Nodes Involved:**  
  - Slack Tool  
  - Email Reply Tool  

- **Node Details:**  
  - **Slack Tool:**  
    - Sends alert messages to a channel named "all-ai-agents-hub"  
    - Input "alertMessage" string generated by Orchestrator agent  
    - Slack OAuth credentials required  

  - **Email Reply Tool:**  
    - Sends apology or final email responses back to customers  
    - Uses reply operation with original email ID for threading  
    - Gmail OAuth credentials required  

- **Edge Cases & Failures:**  
  - Slack API rate limits or invalid channel IDs  
  - Gmail sending restrictions or quota exceeded  

---

#### 2.7 Knowledge Base Loader & Vector Storage

- **Overview:**  
  Supports manual ingestion of corporate policy documents (PDF or copied text) into a Supabase vector database for RAG retrieval.

- **Nodes Involved:**  
  - Upload File / Copy Text here (Form Trigger)  
  - Default Data Loader (PDF loader)  
  - Embeddings OpenAI (embedding generation)  
  - Knowledge Base Storage (Supabase vector DB insert)

- **Node Details:**  
  - **Upload File / Copy Text here:**  
    - Form trigger allowing manual input of policy documents as text or files  
    - Starts ingestion process  

  - **Default Data Loader:**  
    - Uses PDF loader to parse uploaded files into documents  

  - **Embeddings OpenAI:**  
    - Generates embeddings for documents using OpenAI embeddings API  

  - **Knowledge Base Storage:**  
    - Inserts embeddings and document metadata into Supabase vector DB  
    - Requires Supabase API credentials  

- **Edge Cases & Failures:**  
  - Incorrect file format or corrupted files  
  - Embedding generation API errors or limits  
  - Vector DB insertion errors  

---

#### 2.8 Supporting Utilities

- **Overview:**  
  Several utility nodes provide session memory buffers, JSON cleaning code, and placeholders for input schema in sub-workflows.

- **Nodes Involved:**  
  - Simple Memory nodes (for each agent)  
  - Clean JSON / Clean Json code nodes  
  - Placeholder 1, 2, 3 (Set nodes for input schema)  
  - Sticky Notes (documentation and instructions)

- **Node Details:**  
  - Memory nodes maintain session context for conversation continuity  
  - Clean JSON nodes sanitize AI outputs for reliable downstream processing  
  - Placeholder nodes serve as input templates when workflows are copied or used independently  

- **Edge Cases & Failures:**  
  - Memory overflow or session key collisions  
  - Code node errors if unexpected input format encountered  

---

### 3. Summary Table

| Node Name                      | Node Type                          | Functional Role                                      | Input Node(s)                | Output Node(s)                           | Sticky Note                                                                                                                              |
|--------------------------------|----------------------------------|-----------------------------------------------------|-----------------------------|----------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------|
| Gmail Trigger                  | gmailTrigger                     | Incoming email trigger                               |                             | Guardrails                             | ## Gmail trigger  & Guard Rails (Input + Security Layer) Incoming emails are processed here. The Guardrails node (powered by an LLM) scans for PII and prompt injection attacks. If a threat is detected, the workflow stops and logs the incident to Airtable. |
| Guardrails                    | @n8n/n8n-nodes-langchain.guardrails | Security check for PII, prompt injection, profanity | Gmail Trigger               | Orchestrator Agent, Log Threats in Airtable |                                                                                                                                          |
| Log Threats in Airtable       | airtable                        | Logs flagged security incidents                      | Guardrails                  |                                        |                                                                                                                                          |
| Orchestrator Agent            | @n8n/n8n-nodes-langchain.agent  | Central workflow orchestrator, multi-phase logic    | Guardrails                  | Slack Tool, Email Reply Tool, Call Ticket Analyser Agent, Call Knowledge Worker Agent, Call Resolution Agent | ## The Master Orchestrator This is the central "brain." ...                                                                                 |
| Slack Tool                   | slackTool                      | Sends urgent escalation alerts                       | Orchestrator Agent          | Orchestrator Agent                     |                                                                                                                                          |
| Email Reply Tool             | gmailTool                      | Sends email replies to customers                     | Orchestrator Agent          | Orchestrator Agent                     |                                                                                                                                          |
| Call Ticket Analyser Agent   | toolWorkflow                   | Calls ticket triage sub-workflow                     | Orchestrator Agent          | Orchestrator Agent                     |                                                                                                                                          |
| Email / Ticket Analyser Agent| @n8n/n8n-nodes-langchain.agent  | Classifies ticket category, urgency, sentiment       | Call Ticket Analyser Agent  | Clean JSON                            | ## Email / Ticket Analyzer It scans incoming emails to detect sentiment (e.g., Furious) and assigns a priority score. Its JSON output is the critical signal that tells the Master Agent whether to trigger an emergency escalation or proceed with a standard response. |
| Ticket Analyser LLM          | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | LLM model for ticket analysis                        | Email / Ticket Analyser Agent |                                        |                                                                                                                                          |
| Clean JSON                  | code                           | Cleans and parses AI JSON output                      | Email / Ticket Analyser Agent | Orchestrator Agent                    |                                                                                                                                          |
| Simple Memory for Email Analyser Agent | @n8n/n8n-nodes-langchain.memoryBufferWindow | Stores conversation context for ticket analyser      | Email / Ticket Analyser Agent |                                        |                                                                                                                                          |
| Call Knowledge Worker Agent  | toolWorkflow                   | Calls knowledge retrieval sub-workflow               | Orchestrator Agent          | Orchestrator Agent                    |                                                                                                                                          |
| Knowledge worker and Investigator | @n8n/n8n-nodes-langchain.agent  | Retrieves knowledge base facts via vector DB         | Call Knowledge Worker Agent | Clean Json                           | ## Knowledge Loader / Retriever Run this section manually (ONCE) to ingest your Company policy data ...                                  |
| RAG LLM                    | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | LLM model for knowledge retrieval                    | Knowledge worker and Investigator |                                        |                                                                                                                                          |
| Knowledge Base Retrieval     | @n8n/n8n-nodes-langchain.vectorStoreSupabase | Vector DB search tool for company policies           | Knowledge worker and Investigator |                                        |                                                                                                                                          |
| Clean Json                  | code                           | Sanitizes retrieved facts for downstream use         | Knowledge worker and Investigator | Call Resolution Agent                |                                                                                                                                          |
| Simple Memory for RAG agent  | @n8n/n8n-nodes-langchain.memoryBufferWindow | Stores context for RAG agent                          | Knowledge worker and Investigator |                                        |                                                                                                                                          |
| Call Resolution Agent        | toolWorkflow                   | Calls resolution drafting sub-workflow               | Orchestrator Agent          | Orchestrator Agent                    |                                                                                                                                          |
| Resolution Agent            | @n8n/n8n-nodes-langchain.agent  | Drafts empathetic email responses                      | Call Resolution Agent       | Email Reply Tool                     | ## Resolution Writer The specialized "Drafter" for the team. Synthesizes verified facts into polite responses. ...                         |
| Resolution Agent LLM        | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | LLM model for drafting email responses               | Resolution Agent            |                                        |                                                                                                                                          |
| Simple Memory for Resolution Agent | @n8n/n8n-nodes-langchain.memoryBufferWindow | Stores context for Resolution Agent                   | Resolution Agent            |                                        |                                                                                                                                          |
| Upload File / Copy Text here | formTrigger                   | Manual upload or text input for company policy docs |                             | Knowledge Base Storage               | ## Knowledge Loader Run once to ingest company policy data.                                                                               |
| Default Data Loader          | @n8n/n8n-nodes-langchain.documentDefaultDataLoader | Loads PDF documents for ingestion                      | Upload File / Copy Text here | Embeddings OpenAI                   |                                                                                                                                          |
| Embeddings OpenAI           | @n8n/n8n-nodes-langchain.embeddingsOpenAi | Generates vector embeddings for documents             | Default Data Loader         | Knowledge Base Storage, Knowledge Base Retrieval |                                                                                                                                          |
| Knowledge Base Storage       | @n8n/n8n-nodes-langchain.vectorStoreSupabase | Inserts documents and embeddings into vector DB       | Upload File / Copy Text here |                                        |                                                                                                                                          |
| Simple Memory               | @n8n/n8n-nodes-langchain.memoryBufferWindow | Stores context for Orchestrator                       | Gmail Trigger              | Orchestrator Agent                  |                                                                                                                                          |
| Guard LLM                   | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | LLM for guardrails checking                            |                             | Guardrails                         |                                                                                                                                          |
| Reasoning Model for Orchestrator | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | Additional reasoning LLM for orchestrator             |                             | Orchestrator Agent                  |                                                                                                                                          |
| Clean JSON                  | code                           | Cleans AI output from Knowledge Worker                 | Knowledge worker and Investigator | Call Resolution Agent                |                                                                                                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger:**  
   - Node Type: gmailTrigger  
   - Configure OAuth2 credentials for Gmail  
   - Set filter to unread emails from the desired sender or email address  
   - Poll every minute  

2. **Add Guardrails Node:**  
   - Type: LangChain Guardrails node  
   - Use Google Gemini LLM as Guard LLM with hate speech safety settings  
   - Configure custom guardrails: PII detection (all types), prompt injection check (fail/pass), profanity/hate speech detection  
   - Input text: set to `{{$json.snippet}}` from Gmail Trigger  
   - Connect Gmail Trigger → Guardrails  

3. **Add Airtable Logging Node:**  
   - Type: airtable (upsert operation)  
   - Configure Airtable personal access token credentials  
   - Set Base and Table to your threat logging base/table  
   - Map email ID and sender email as keys  
   - Connect Guardrails (fail output) → Log Threats in Airtable  
   - Guardrails (pass output) → Orchestrator Agent  

4. **Create Orchestrator Agent Node:**  
   - Type: LangChain Agent  
   - Use Google Gemini LLM (Reasoning Model for Orchestrator) with your API credentials  
   - Input text: `{{$('Gmail Trigger').item.json.snippet}}`  
   - Provide detailed system message defining the 5-phase process (triage, evaluation, knowledge retrieval, resolution drafting, final sending) and strict tool calling rules  
   - Add Simple Memory node with sessionKey as email ID connected as AI memory  
   - Connect Guardrails (pass output) → Orchestrator Agent  

5. **Add Slack Tool Node:**  
   - Type: slackTool  
   - Configure Slack OAuth credentials  
   - Set channel to your escalation alert channel ID  
   - Input text to be provided by Orchestrator agent via alertMessage parameter  
   - Connect Orchestrator Agent (Slack tool output) → Slack Tool  

6. **Add Email Reply Tool Node:**  
   - Type: gmailTool (reply operation)  
   - Use Gmail OAuth2 credentials  
   - Configure to reply only to sender, message body from Orchestrator agent output  
   - Connect Orchestrator Agent (Email reply output) → Email Reply Tool  

7. **Create Call Ticket Analyser Agent Node:**  
   - Type: toolWorkflow  
   - Set workflow ID to your separate "Ticket Analyser" sub-workflow  
   - Map inputs: query from email snippet, threadId, senderEmail  
   - Connect Orchestrator Agent (call Ticket Analyser output) → Call Ticket Analyser Agent  

8. **Build Ticket Analyser Sub-Workflow:**  
   - Input placeholder Set node for expected input fields  
   - Simple Memory node keyed by senderEmail + threadId  
   - Email / Ticket Analyser Agent (LangChain Agent) with Gemini LLM  
   - Clean JSON code node to parse and sanitize output  
   - Connect nodes in sequence and return structured JSON  

9. **Create Call Knowledge Worker Agent Node:**  
   - Type: toolWorkflow  
   - Set workflow ID to your separate "Knowledge Worker & Investigation Agent" sub-workflow  
   - Map inputs: query, threadId, senderEmail, triageSummary, triageCategory  
   - Connect Orchestrator Agent → Call Knowledge Worker Agent  

10. **Build Knowledge Worker Sub-Workflow:**  
    - Input placeholder Set node for inputs  
    - Simple Memory node for session context  
    - Knowledge worker and Investigator LangChain Agent using Gemini LLM  
    - Knowledge Base Retrieval node querying Supabase vector DB (n8n_documents table)  
    - Clean Json code node to sanitize facts field  
    - Connect nodes appropriately  

11. **Create Call Resolution Agent Node:**  
    - Type: toolWorkflow  
    - Set workflow ID to your separate "Resolution Agent" sub-workflow  
    - Map inputs: ragFacts (retrieved facts), customerQuery, triageSummary  
    - Connect Orchestrator Agent → Call Resolution Agent  

12. **Build Resolution Agent Sub-Workflow:**  
    - Input placeholder Set node for inputs  
    - Simple Memory node for session context  
    - Resolution Agent LangChain Agent with Gemini LLM  
    - Configure system message to draft empathetic email replies only (no sending)  
    - Connect nodes accordingly  

13. **Set Up Knowledge Base Loader Workflow:**  
    - Form Trigger node allowing manual upload or text input  
    - Default Data Loader node configured with PDF loader  
    - Embeddings OpenAI node for vector embeddings generation  
    - Knowledge Base Storage node inserting embeddings into Supabase vector DB  
    - Ensure Supabase and OpenAI API credentials are configured  

14. **Configure Credentials:**  
    - Gmail OAuth2 (read and send scopes)  
    - Slack OAuth for posting messages  
    - Google Gemini (PaLM) API for LLM nodes  
    - OpenAI API for embeddings  
    - Airtable Personal Access Token  
    - Supabase API key  

15. **Connect All Nodes:**  
    - Gmail Trigger → Guardrails → (fail) Log Threats / (pass) Orchestrator Agent  
    - Orchestrator Agent calls Ticket Analyser, Knowledge Worker, Resolution Agent sub-workflows via toolWorkflow nodes  
    - Orchestrator Agent triggers Slack Tool and Email Reply Tool as needed based on logic  
    - Memory nodes attached to corresponding agents  
    - Clean JSON/code nodes sanitize AI outputs before further processing  

16. **Testing:**  
    - Send test emails to trigger workflow  
    - Verify escalation paths (Slack alerts and apology emails)  
    - Verify knowledge retrieval and draft email responses for standard queries  
    - Monitor Airtable logs for threat detections  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                          | Context or Link                                                                                              |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| The Master Orchestrator is the central brain maintaining state and enforcing protocol, including escalation rules and multi-agent coordination.                                                                                                       | Sticky Note1                                                                                                 |
| Incoming emails are first processed by Guardrails powered by an LLM to scan for PII, prompt injections, and profanity, ensuring security compliance before further processing.                                                                         | Sticky Note3                                                                                                 |
| The Email / Ticket Analyzer workflow analyzes customer message sentiment and priority to decide escalation or standard processing.                                                                                                                    | Sticky Note                                                                                                   |
| Knowledge Loader workflow is used once to ingest corporate policy documents into the vector DB for use by the Knowledge Worker.                                                                                                                     | Sticky Note5                                                                                                 |
| The Resolution Writer drafts final email replies using verified facts and customer query, maintaining a human-in-the-loop approach by not sending emails automatically itself.                                                                       | Sticky Note8                                                                                                 |
| Multi-Agent Customer Support Orchestrator overview describing the entire tier 2 AI-powered customer support process, including setup instructions and testing steps.                                                                                   | Sticky Note14                                                                                                |
| For detailed policy documents ingestion, the Upload File / Copy Text here form trigger provides an interface to add or update company policies used by RAG system.                                                                                    | Upload File / Copy Text here node                                                                            |
| The workflow relies heavily on Google Gemini (PaLM) LLM API for reasoning, guardrails, triage, knowledge retrieval, and resolution drafting, along with OpenAI for embeddings, and Supabase as vector store.                                           | Credentials configuration                                                                                     |
| Slack channel used for escalation alerts is “all-ai-agents-hub” with channel ID C0900FG5BPC.                                                                                                                                                            | Slack Tool node configuration                                                                                 |
| Apology template and escalation protocols are strictly enforced by the Orchestrator Agent via system messages and tool calling logic.                                                                                                                | Orchestrator Agent system message details                                                                    |
| The workflow is modular with sub-workflows for each agent, allowing easy maintenance and updates per component. Sub-workflow placeholders must be replaced by “When executed by another workflow trigger” nodes for standalone use.                  | Multiple sticky notes and instructions in nodes                                                             |
| All AI-generated outputs are sanitized via code nodes to prevent malformed JSON or undesirable text artifacts from breaking workflow logic.                                                                                                          | Clean JSON and Clean Json code nodes                                                                          |
| The workflow uses session keys based on email sender and thread ID to maintain conversational context across memory buffer nodes for consistent AI performance.                                                                                       | Simple Memory nodes configuration                                                                             |

---

**Disclaimer:**  
The provided workflow content originates exclusively from an automated n8n workflow designed with strict adherence to content policies, containing no illegal, offensive, or protected elements. All data handled is lawful and publicly accessible.
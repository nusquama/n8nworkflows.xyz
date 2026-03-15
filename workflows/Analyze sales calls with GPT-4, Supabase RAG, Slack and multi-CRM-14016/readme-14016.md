Analyze sales calls with GPT-4, Supabase RAG, Slack and multi-CRM

https://n8nworkflows.xyz/workflows/analyze-sales-calls-with-gpt-4--supabase-rag--slack-and-multi-crm-14016


# Analyze sales calls with GPT-4, Supabase RAG, Slack and multi-CRM

# 1. Workflow Overview

This workflow, titled **Autonomous AI Sales Coach & Evaluator**, is designed to ingest sales meeting data, retrieve the meeting transcript, normalize the payload, and pass the result to an AI agent that evaluates the call using GPT-4-style chat capabilities plus external business context from CRM systems and a Supabase vector store. The generated feedback is then distributed through Slack and email.

Typical use cases include:
- Automated post-call coaching for sales representatives
- Call quality scoring and evaluation
- CRM-enriched call analysis across multiple sales systems
- Retrieval-augmented feedback using historical/internal knowledge stored in Supabase

The workflow can be divided into the following logical blocks.

## 1.1 Input Reception and Transcript Retrieval
The workflow starts from a webhook that receives meeting details, then performs an HTTP request to fetch the transcript or related meeting content.

## 1.2 Payload Cleaning and Participant Classification
A Code node reshapes the incoming data into a cleaner JSON structure and likely distinguishes internal participants from prospects/customers using an email domain rule.

## 1.3 AI Analysis Orchestration
An AI Agent receives the cleaned call data and uses:
- an OpenAI chat model for reasoning,
- memory for context continuity,
- CRM tools for contextual lookups,
- a Supabase vector store for retrieval-augmented knowledge access.

## 1.4 Feedback Delivery
The AI agent’s output is delivered to Slack and by email.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and Transcript Retrieval

### Overview
This block receives the meeting event payload and then retrieves the corresponding transcript or meeting details from an external system. It is the workflow’s entry point and provides the raw content needed for downstream analysis.

### Nodes Involved
- Getting Meeting Details
- Extracting Transcript

### Node Details

#### 2.1.1 Getting Meeting Details
- **Type and technical role:** `n8n-nodes-base.webhook`  
  Entry-point trigger node that exposes an HTTP endpoint for incoming meeting data.
- **Configuration choices:**  
  The JSON shows no explicit custom parameters, so this is likely using default webhook behavior. In practice, this means the endpoint accepts an HTTP request and emits the payload into the workflow.
- **Key expressions or variables used:**  
  None visible in the JSON.
- **Input and output connections:**  
  - Input: none
  - Output: `Extracting Transcript`
- **Version-specific requirements:**  
  `typeVersion: 2.1`
- **Edge cases or potential failure types:**  
  - Webhook not activated or wrong URL used by upstream source
  - Invalid payload structure
  - Missing meeting identifier needed by the next HTTP request
  - Authentication/signature validation not implemented if expected by source platform
- **Sub-workflow reference:**  
  None

#### 2.1.2 Extracting Transcript
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Makes an outbound HTTP request to retrieve transcript data or expanded meeting metadata.
- **Configuration choices:**  
  Parameters are empty in the provided JSON, so the specific method, URL, headers, auth, and body are not visible. The functional role strongly implies this node uses values from the webhook payload to fetch the transcript from a meeting provider, recording platform, or transcription API.
- **Key expressions or variables used:**  
  Not visible, but this node almost certainly depends on fields from `Getting Meeting Details`, such as meeting ID, recording ID, or transcript URL.
- **Input and output connections:**  
  - Input: `Getting Meeting Details`
  - Output: `Cleaned JSON Payload`
- **Version-specific requirements:**  
  `typeVersion: 4.3`
- **Edge cases or potential failure types:**  
  - Missing transcript ID or URL
  - API authentication failure
  - 404 if transcript is not yet available
  - Rate limits or timeout on transcript provider
  - Non-JSON response causing downstream parsing issues
- **Sub-workflow reference:**  
  None

---

## 2.2 Payload Cleaning and Participant Classification

### Overview
This block transforms the raw transcript response into a clean payload for AI processing. It also contains business logic to distinguish internal company attendees from external participants based on email domain matching.

### Nodes Involved
- Cleaned JSON Payload

### Node Details

#### 2.2.1 Cleaned JSON Payload
- **Type and technical role:** `n8n-nodes-base.code`  
  Executes custom JavaScript to clean, normalize, and prepare the transcript payload for the AI agent.
- **Configuration choices:**  
  The actual code is not included in the JSON, but the node note explicitly indicates that the script contains a variable:
  - `const internalDomain = "@yourcompany.com";`
  
  This should be replaced with the company’s real internal email domain. The purpose is to classify attendees and infer who is the sales rep versus the customer/prospect.
- **Key expressions or variables used:**  
  - `internalDomain`
- **Input and output connections:**  
  - Input: `Extracting Transcript`
  - Output: `AI Agent`
- **Version-specific requirements:**  
  `typeVersion: 2`
- **Edge cases or potential failure types:**  
  - Misclassification if the company domain is not updated
  - Misclassification if both internal and external participants use the same public domain such as `@gmail.com`
  - JavaScript runtime errors if expected payload fields are missing
  - Null/undefined transcript sections leading to malformed AI prompts
- **Sub-workflow reference:**  
  None

**Embedded note associated with this node:**  
“In the code above, look for line 8: `const internalDomain = "@yourcompany.com";` Change `@yourcompany.com` to your actual business domain. Note: If your team uses standard free `@gmail.com` addresses instead of a custom business domain, you can set it to `@gmail.com`. However, be aware that if a client also joins using a personal Gmail account, the system might mislabel them as a Sales Rep.”

---

## 2.3 AI Analysis Orchestration

### Overview
This is the core intelligence block. The AI agent uses an OpenAI chat model, memory, CRM lookup tools, and a Supabase vector store with embeddings to generate contextual sales-call feedback.

### Nodes Involved
- AI Agent
- OpenAI Chat Model
- Simple Memory
- Get many deals in HubSpot
- Get many deals in Pipedrive
- Get many leads in Salesforce
- Supabase Vector Store
- Embeddings OpenAI

### Node Details

#### 2.3.1 AI Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  Central orchestration node that accepts the cleaned meeting payload and invokes language model reasoning plus connected tools.
- **Configuration choices:**  
  Parameters are not visible, but based on connections this agent is configured with:
  - one chat model,
  - one memory input,
  - multiple tool integrations,
  - main input from the cleaned payload.
- **Key expressions or variables used:**  
  Not visible in JSON. Likely consumes normalized transcript fields from `Cleaned JSON Payload`.
- **Input and output connections:**  
  - Main input: `Cleaned JSON Payload`
  - AI language model: `OpenAI Chat Model`
  - AI memory: `Simple Memory`
  - AI tools: `Get many deals in HubSpot`, `Get many deals in Pipedrive`, `Get many leads in Salesforce`, `Supabase Vector Store`
  - Main outputs: `Send the Feedback`, `Email the Feedback`
- **Version-specific requirements:**  
  `typeVersion: 3`
- **Edge cases or potential failure types:**  
  - Missing model credentials
  - Tool invocation failure from CRM or vector store
  - Token/context overflow if transcript is long
  - Inconsistent tool outputs causing reasoning issues
  - Prompting gaps if required instructions were configured manually and are incomplete
- **Sub-workflow reference:**  
  None

#### 2.3.2 OpenAI Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Provides the language model used by the AI Agent for analysis and feedback generation.
- **Configuration choices:**  
  Exact model name, temperature, and token limits are not visible. Given the workflow title/description, this is intended for GPT-4-class analysis.
- **Key expressions or variables used:**  
  None visible.
- **Input and output connections:**  
  - Output: connected to `AI Agent` via `ai_languageModel`
- **Version-specific requirements:**  
  `typeVersion: 1.3`
- **Edge cases or potential failure types:**  
  - Invalid OpenAI credentials
  - Model unavailable in the account
  - Rate limiting
  - High token usage due to long transcripts and CRM/RAG context
- **Sub-workflow reference:**  
  None

#### 2.3.3 Simple Memory
- **Type and technical role:** `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
  Supplies short-term conversational memory to the AI Agent.
- **Configuration choices:**  
  No explicit parameters are shown. This likely stores a limited context window for agent reasoning during the execution.
- **Key expressions or variables used:**  
  None visible.
- **Input and output connections:**  
  - Output: connected to `AI Agent` via `ai_memory`
- **Version-specific requirements:**  
  `typeVersion: 1.3`
- **Edge cases or potential failure types:**  
  - Memory not especially useful in a single-pass workflow unless the agent performs iterative tool use
  - Excessive memory window could increase prompt size
- **Sub-workflow reference:**  
  None

#### 2.3.4 Get many deals in HubSpot
- **Type and technical role:** `n8n-nodes-base.hubspotTool`  
  Exposes HubSpot deal retrieval as an AI-callable tool.
- **Configuration choices:**  
  Parameters are not visible. Based on the node name, this tool is configured to retrieve multiple deals from HubSpot, likely for company/contact/opportunity context.
- **Key expressions or variables used:**  
  None visible directly. The AI agent likely passes search terms or filters.
- **Input and output connections:**  
  - Output: connected to `AI Agent` via `ai_tool`
- **Version-specific requirements:**  
  `typeVersion: 2.2`
- **Edge cases or potential failure types:**  
  - HubSpot credential issues
  - Portal permissions insufficient for deals
  - Query returning too many records or no records
  - Ambiguous contact/company matching
- **Sub-workflow reference:**  
  None

#### 2.3.5 Get many deals in Pipedrive
- **Type and technical role:** `n8n-nodes-base.pipedriveTool`  
  Exposes Pipedrive deal retrieval as an AI-callable tool.
- **Configuration choices:**  
  Parameters are not shown; likely configured to search or list deals relevant to the meeting participants/account.
- **Key expressions or variables used:**  
  None visible.
- **Input and output connections:**  
  - Output: connected to `AI Agent` via `ai_tool`
- **Version-specific requirements:**  
  `typeVersion: 1`
- **Edge cases or potential failure types:**  
  - API token or OAuth credential failures
  - Permissions mismatch
  - Large result sets
  - Duplicate or similarly named organizations creating incorrect context
- **Sub-workflow reference:**  
  None

#### 2.3.6 Get many leads in Salesforce
- **Type and technical role:** `n8n-nodes-base.salesforceTool`  
  Exposes Salesforce lead retrieval as an AI-callable tool.
- **Configuration choices:**  
  Based on the node name, this tool is focused on lead lookup rather than opportunities/accounts. Exact object and query settings are not visible.
- **Key expressions or variables used:**  
  None visible.
- **Input and output connections:**  
  - Output: connected to `AI Agent` via `ai_tool`
- **Version-specific requirements:**  
  `typeVersion: 1`
- **Edge cases or potential failure types:**  
  - Salesforce authentication/session issues
  - SOQL/query constraints
  - Wrong object choice if data lives in Contacts/Opportunities instead of Leads
  - Result ambiguity for common names or email aliases
- **Sub-workflow reference:**  
  None

#### 2.3.7 Supabase Vector Store
- **Type and technical role:** `@n8n/n8n-nodes-langchain.vectorStoreSupabase`  
  Makes a Supabase-backed vector index available as an AI tool for semantic retrieval.
- **Configuration choices:**  
  The node is wired as an AI tool and depends on `Embeddings OpenAI` for embedding generation. Specific table, query mode, metadata filters, and similarity settings are not visible.
- **Key expressions or variables used:**  
  None visible.
- **Input and output connections:**  
  - Embedding input: `Embeddings OpenAI`
  - Tool output: connected to `AI Agent` via `ai_tool`
- **Version-specific requirements:**  
  `typeVersion: 1.3`
- **Edge cases or potential failure types:**  
  - Supabase URL/key issues
  - Missing pgvector setup or mismatched schema
  - Empty knowledge base returns low-value context
  - Embedding-model mismatch with indexed vectors
  - Retrieval noise if chunking/metadata strategy was poor during ingestion
- **Sub-workflow reference:**  
  None

#### 2.3.8 Embeddings OpenAI
- **Type and technical role:** `@n8n/n8n-nodes-langchain.embeddingsOpenAi`  
  Supplies embedding generation for semantic search against Supabase.
- **Configuration choices:**  
  Exact embedding model is not visible.
- **Key expressions or variables used:**  
  None visible.
- **Input and output connections:**  
  - Output: connected to `Supabase Vector Store` via `ai_embedding`
- **Version-specific requirements:**  
  `typeVersion: 1.2`
- **Edge cases or potential failure types:**  
  - Invalid OpenAI credentials
  - Embedding model unavailable or changed
  - Incompatibility with vectors previously indexed using another embedding model
- **Sub-workflow reference:**  
  None

---

## 2.4 Feedback Delivery

### Overview
This block sends the AI-generated evaluation to communication channels. The workflow currently distributes output through both Slack and email in parallel.

### Nodes Involved
- Email the Feedback
- Send the Feedback

### Node Details

#### 2.4.1 Email the Feedback
- **Type and technical role:** `n8n-nodes-base.emailSend`  
  Sends the AI-generated feedback via email.
- **Configuration choices:**  
  Parameters are not visible. Expected configuration includes recipient, subject, body, and SMTP or email transport credentials.
- **Key expressions or variables used:**  
  Not visible, but likely references the AI Agent output text.
- **Input and output connections:**  
  - Input: `AI Agent`
  - Output: none
- **Version-specific requirements:**  
  `typeVersion: 2.1`
- **Edge cases or potential failure types:**  
  - SMTP or mail transport credential errors
  - Invalid recipient address
  - HTML/text formatting problems if output is complex
  - Large message body if transcript excerpts are included
- **Sub-workflow reference:**  
  None

#### 2.4.2 Send the Feedback
- **Type and technical role:** `n8n-nodes-base.slack`  
  Sends the AI-generated feedback to Slack.
- **Configuration choices:**  
  Parameters are not shown. Typical setup would specify channel, message text/blocks, and Slack credentials.
- **Key expressions or variables used:**  
  Not visible, but likely maps message content from `AI Agent`.
- **Input and output connections:**  
  - Input: `AI Agent`
  - Output: none
- **Version-specific requirements:**  
  `typeVersion: 2.4`
- **Edge cases or potential failure types:**  
  - Slack auth/token issues
  - Bot missing access to target channel
  - Message length or formatting issues
  - Use of Block Kit fields not properly structured if configured dynamically
- **Sub-workflow reference:**  
  None

---

## 2.5 Sticky Notes and Documentation Nodes

### Overview
The workflow contains several sticky-note nodes, but their `content` fields are empty in the provided JSON. They appear to be layout/documentation placeholders rather than active logic.

### Nodes Involved
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4
- Sticky Note5

### Node Details

#### 2.5.1 Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Visual annotation node.
- **Configuration choices:** Empty content.
- **Connections:** None
- **Version:** `1`
- **Edge cases:** None
- **Sub-workflow reference:** None

#### 2.5.2 Sticky Note1
- Same role as above; empty content; no connections; `typeVersion: 1`

#### 2.5.3 Sticky Note2
- Same role as above; empty content; no connections; `typeVersion: 1`

#### 2.5.4 Sticky Note3
- Same role as above; empty content; no connections; `typeVersion: 1`

#### 2.5.5 Sticky Note4
- Same role as above; empty content; no connections; `typeVersion: 1`

#### 2.5.6 Sticky Note5
- Same role as above; empty content; no connections; `typeVersion: 1`

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Getting Meeting Details | Webhook | Receives inbound meeting payload |  | Extracting Transcript |  |
| Extracting Transcript | HTTP Request | Fetches transcript or expanded meeting data | Getting Meeting Details | Cleaned JSON Payload |  |
| Cleaned JSON Payload | Code | Normalizes payload and classifies internal/external attendees | Extracting Transcript | AI Agent | In the code above, look for line 8: `const internalDomain = "@yourcompany.com";` |
| Cleaned JSON Payload | Code | Normalizes payload and classifies internal/external attendees | Extracting Transcript | AI Agent | Change `@yourcompany.com` to your actual business domain. |
| Cleaned JSON Payload | Code | Normalizes payload and classifies internal/external attendees | Extracting Transcript | AI Agent | Note: If your team uses standard free `@gmail.com` addresses instead of a custom business domain, you can set it to `@gmail.com`. However, be aware that if a client also joins using a personal Gmail account, the system might mislabel them as a Sales Rep. |
| AI Agent | LangChain Agent | Performs sales-call analysis using model, memory, CRM tools, and vector retrieval | Cleaned JSON Payload | Send the Feedback, Email the Feedback |  |
| OpenAI Chat Model | OpenAI Chat Model | Provides LLM reasoning for the agent |  | AI Agent |  |
| Simple Memory | Buffer Window Memory | Supplies agent memory context |  | AI Agent |  |
| Get many deals in HubSpot | HubSpot Tool | AI-accessible HubSpot deal lookup |  | AI Agent |  |
| Get many deals in Pipedrive | Pipedrive Tool | AI-accessible Pipedrive deal lookup |  | AI Agent |  |
| Get many leads in Salesforce | Salesforce Tool | AI-accessible Salesforce lead lookup |  | AI Agent |  |
| Supabase Vector Store | Supabase Vector Store | AI-accessible semantic retrieval over knowledge base | Embeddings OpenAI | AI Agent |  |
| Embeddings OpenAI | OpenAI Embeddings | Generates embeddings for vector retrieval |  | Supabase Vector Store |  |
| Email the Feedback | Send Email | Emails the generated evaluation | AI Agent |  |  |
| Send the Feedback | Slack | Posts the generated evaluation to Slack | AI Agent |  |  |
| Sticky Note | Sticky Note | Visual annotation placeholder |  |  |  |
| Sticky Note1 | Sticky Note | Visual annotation placeholder |  |  |  |
| Sticky Note2 | Sticky Note | Visual annotation placeholder |  |  |  |
| Sticky Note3 | Sticky Note | Visual annotation placeholder |  |  |  |
| Sticky Note4 | Sticky Note | Visual annotation placeholder |  |  |  |
| Sticky Note5 | Sticky Note | Visual annotation placeholder |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

Below is a step-by-step rebuild process based on the available workflow JSON. Because several nodes have empty exported parameters, some configuration must be inferred and completed with your own system-specific values.

1. **Create a new workflow**
   - Name it: **Autonomous AI Sales Coach & Evaluator**
   - Keep execution order at the default compatible mode if needed; the JSON uses `executionOrder: v1`.

2. **Add a Webhook node**
   - Node name: **Getting Meeting Details**
   - Type: **Webhook**
   - Use it as the workflow trigger.
   - Configure:
     - HTTP method according to your meeting platform’s webhook requirements, typically `POST`
     - Response mode as needed, often immediate acknowledgment
     - Optional path name for the endpoint
   - Ensure the upstream meeting system sends meeting metadata such as:
     - meeting ID
     - transcript ID or recording reference
     - attendees
     - timestamps
     - call title or account name

3. **Add an HTTP Request node**
   - Node name: **Extracting Transcript**
   - Type: **HTTP Request**
   - Connect: `Getting Meeting Details -> Extracting Transcript`
   - Configure:
     - Method: likely `GET` or `POST`, depending on the transcript provider API
     - URL: use an expression based on webhook data if transcript endpoint is dynamic
     - Authentication: API key, bearer token, or OAuth2 depending on provider
     - Response format: JSON
   - Example expected use:
     - fetch transcript text
     - fetch speaker turns
     - fetch meeting summary metadata

4. **Add a Code node**
   - Node name: **Cleaned JSON Payload**
   - Type: **Code**
   - Connect: `Extracting Transcript -> Cleaned JSON Payload`
   - In the code:
     - parse the HTTP response
     - normalize transcript structure
     - extract meeting name, attendees, speaker labels, timestamps, and transcript text
     - identify internal vs external participants
   - Add or update this constant:
     - `const internalDomain = "@yourcompany.com";`
   - Replace the sample domain with your actual business domain.
   - Produce a clean JSON object for the AI agent, such as:
     - meeting title
     - rep name
     - customer name
     - transcript
     - objections
     - call date
     - account/company identifiers if available

5. **Add an AI Agent node**
   - Node name: **AI Agent**
   - Type: **AI Agent** / LangChain Agent
   - Connect: `Cleaned JSON Payload -> AI Agent`
   - Configure the agent prompt to instruct it to:
     - analyze the sales call
     - score or evaluate rep performance
     - use CRM tools when account/deal context is relevant
     - use the vector store for company knowledge, playbooks, or sales methodology
     - return structured feedback suitable for Slack and email
   - If supported, define expected output sections such as:
     - overall score
     - strengths
     - weaknesses
     - objections handled
     - missed opportunities
     - coaching recommendations

6. **Add an OpenAI Chat Model node**
   - Node name: **OpenAI Chat Model**
   - Type: **OpenAI Chat Model**
   - Connect it to the AI Agent’s `ai_languageModel` port
   - Configure:
     - OpenAI credentials
     - GPT-4-class model or equivalent available in your account
     - temperature low to moderate for consistent evaluations
     - max tokens sized for transcript analysis

7. **Add a Memory node**
   - Node name: **Simple Memory**
   - Type: **Buffer Window Memory**
   - Connect it to the AI Agent’s `ai_memory` port
   - Configure a small memory window unless you expect multi-step agent/tool interactions

8. **Add a HubSpot tool node**
   - Node name: **Get many deals in HubSpot**
   - Type: **HubSpot Tool**
   - Connect it to the AI Agent’s `ai_tool` port
   - Configure HubSpot credentials
   - Set the tool operation to retrieve multiple deals
   - Expose searchable fields relevant to the meeting, such as:
     - company name
     - contact email
     - deal owner
     - pipeline stage

9. **Add a Pipedrive tool node**
   - Node name: **Get many deals in Pipedrive**
   - Type: **Pipedrive Tool**
   - Connect it to the AI Agent’s `ai_tool` port
   - Configure Pipedrive credentials
   - Set it to search or retrieve multiple deals matching meeting context

10. **Add a Salesforce tool node**
    - Node name: **Get many leads in Salesforce**
    - Type: **Salesforce Tool**
    - Connect it to the AI Agent’s `ai_tool` port
    - Configure Salesforce credentials
    - Set the operation to retrieve multiple lead records
    - If your organization stores prospects in Contacts or Opportunities instead of Leads, adjust accordingly

11. **Add an OpenAI Embeddings node**
    - Node name: **Embeddings OpenAI**
    - Type: **OpenAI Embeddings**
    - Configure OpenAI credentials
    - Choose the same embedding family used to build your vector index

12. **Add a Supabase Vector Store node**
    - Node name: **Supabase Vector Store**
    - Type: **Supabase Vector Store**
    - Connect:
      - `Embeddings OpenAI -> Supabase Vector Store` via embedding connection
      - `Supabase Vector Store -> AI Agent` via `ai_tool`
    - Configure:
      - Supabase URL
      - Supabase API/service credentials
      - vector table or document table
      - content column
      - embedding column
      - metadata columns if used
    - Ensure your Supabase instance supports pgvector and already contains indexed knowledge documents

13. **Populate the vector knowledge base**
    - Before relying on retrieval, load relevant sales documents into Supabase, for example:
      - sales playbooks
      - objection handling guides
      - product positioning notes
      - pricing and packaging information
      - competitor battle cards
    - Make sure the same embedding model is used for indexing and querying

14. **Add an Email Send node**
    - Node name: **Email the Feedback**
    - Type: **Send Email**
    - Connect: `AI Agent -> Email the Feedback`
    - Configure:
      - SMTP or email service credentials
      - recipient list
      - sender address
      - subject line, e.g. “Sales Call Evaluation – {{$json.meetingTitle}}”
      - body using the AI Agent output
    - Prefer plain text or simple HTML unless your AI output is already formatted safely

15. **Add a Slack node**
    - Node name: **Send the Feedback**
    - Type: **Slack**
    - Connect: `AI Agent -> Send the Feedback`
    - Configure:
      - Slack credentials
      - channel ID or name
      - message body mapped from the AI output
    - If using rich formatting, keep block structures within Slack limits

16. **Optional: add sticky notes**
    - Add visual sticky notes if you want documentation in the canvas
    - In the provided workflow, all sticky note contents are empty, so they are not functionally required

17. **Credential setup checklist**
    - Configure credentials for:
      - OpenAI chat
      - OpenAI embeddings
      - transcript provider API used in HTTP Request
      - HubSpot
      - Pipedrive
      - Salesforce
      - Supabase
      - Slack
      - SMTP/email service

18. **Test the trigger and transcript retrieval**
    - Send a sample webhook payload
    - Confirm `Extracting Transcript` returns valid JSON
    - Confirm `Cleaned JSON Payload` emits a compact, structured object

19. **Test the AI path**
    - Verify the AI Agent can:
      - receive transcript text
      - call the CRM tools successfully
      - call the Supabase vector store
      - return a coherent evaluation

20. **Test delivery channels**
    - Ensure the Slack message posts successfully
    - Ensure email is delivered and formatted correctly

21. **Harden for production**
    - Add error handling if desired, such as:
      - transcript unavailable fallback
      - CRM tool failures tolerated without aborting whole workflow
      - retries for HTTP and Slack/email nodes
      - guardrails for missing participant emails or transcript text

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The Code node contains a domain-based participant classification rule that must be customized before production use. | Applies to internal vs external attendee detection |
| If your company uses public email domains such as Gmail, participant role detection may become unreliable. | Applies to `Cleaned JSON Payload` |
| Several nodes have empty exported parameters in the provided JSON, so endpoint URLs, prompts, credential mappings, and message templates must be completed manually. | Applies to the full workflow |
| The workflow includes a single entry point: the Webhook node `Getting Meeting Details`. | Architecture note |
| The workflow does not invoke any sub-workflows. | Architecture note |
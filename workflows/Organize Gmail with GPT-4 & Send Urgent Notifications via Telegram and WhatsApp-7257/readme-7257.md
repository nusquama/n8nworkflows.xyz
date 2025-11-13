Organize Gmail with GPT-4 & Send Urgent Notifications via Telegram and WhatsApp

https://n8nworkflows.xyz/workflows/organize-gmail-with-gpt-4---send-urgent-notifications-via-telegram-and-whatsapp-7257


# Organize Gmail with GPT-4 & Send Urgent Notifications via Telegram and WhatsApp

### 1. Workflow Overview

This workflow automates the organization of incoming Gmail emails by leveraging GPT-4-based AI agents to classify emails with domain-specific labels and evaluate their urgency. It also sends urgent email notifications via Telegram and WhatsApp for immediate awareness.

**Target Use Cases:**  
- Automatic categorization of incoming emails into Gmail labels based on content, sender, and metadata.  
- Prioritization of emails by urgency with brief summaries.  
- Prompt delivery of urgent email alerts on messaging platforms (Telegram and WhatsApp).  
- Streamlined inbox management for busy users or teams.

**Logical Blocks:**  
- **1.1 Input Reception:** Detect new emails via Gmail Trigger and fetch complete email data.  
- **1.2 AI-Based Email Classification:** Use an AI agent to assign the most appropriate Gmail label ID for email categorization.  
- **1.3 Label Management and Application:** Create new labels if missing, retrieve existing labels, filter correct label ID, and apply to email threads.  
- **1.4 AI-Based Urgency Detection and Summarization:** Independently analyze email urgency and generate a concise summary.  
- **1.5 Urgent Notification Dispatch:** If email classified as urgent, send notifications via Telegram and WhatsApp.  
- **1.6 Support Nodes:** Utilities such as JSON parsing, filtering, batching, waiting, and error handling.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Triggered on new Gmail messages; retrieves full email details for processing.  
- **Nodes Involved:**  
  - Gmail Trigger1  
  - Get a message1  
  - Sticky Note1, Sticky Note2 (documentation nodes)

- **Node Details:**  
  - **Gmail Trigger1**  
    - Type: Gmail Trigger  
    - Role: Watches Gmail inbox for new emails every minute.  
    - Config: Poll every minute, no filter on email metadata.  
    - Inputs: None (trigger node)  
    - Outputs: Email metadata including message ID.  
    - Edge cases: Missed emails if polling delayed; Gmail API quota limits.  
  - **Get a message1**  
    - Type: Gmail node (get message)  
    - Role: Retrieves full email content (body, subject, headers) using message ID from trigger.  
    - Config: Uses message ID from trigger node output JSON.  
    - Inputs: Message ID from Gmail Trigger1.  
    - Outputs: Complete email JSON.  
    - Edge cases: API errors, message deleted between trigger and fetch.  
  - **Sticky Note1 & Sticky Note2**  
    - Type: Sticky Note (documentation)  
    - Role: Provide visual annotations explaining trigger and fetch step.  
    - No inputs/outputs.

#### 1.2 AI-Based Email Classification

- **Overview:** Uses an AI agent to classify email into domain-specific labels by analyzing sender, content, metadata, and prior history.  
- **Nodes Involved:**  
  - AI Agent3  
  - OpenRouter Chat Model2  
  - Google Gemini Chat Model2  
  - Check Sent2  
  - Get a message1 (input to AI Agent3)  
  - Structured Output Parser1  
  - Sticky Note3

- **Node Details:**  
  - **AI Agent3**  
    - Type: LangChain AI agent  
    - Role: Receives full email JSON and runs classification prompt to output label ID only.  
    - Config: Uses OpenRouter Chat Model2 and Google Gemini Chat Model2 as language models with fallback.  
    - Inputs: Email full content from Get a message1, Sent label check nodes for context.  
    - Outputs: JSON with label ID and label name.  
    - Expressions: Complex prompt including email metadata, headers, heuristics.  
    - Edge cases: AI response missing or invalid format; fallback models activated on failure.  
  - **OpenRouter Chat Model2 & Google Gemini Chat Model2**  
    - Type: Language model nodes  
    - Role: Provide language model inference for classification.  
    - Config: OpenRouter model "deepseek-chat-v3-0324:free"; Google Gemini default.  
    - Inputs: Prompt from AI Agent3.  
    - Outputs: AI-generated classification string.  
    - Edge cases: API limits, latency, errors.  
  - **Check Sent2**  
    - Type: GmailTool node (search)  
    - Role: Checks if email was sent to the sender address to enrich classification context.  
    - Config: Query "to:{{ $fromAI('email') }}" with label "SENT".  
    - Inputs: Email sender from AI Agent3 context.  
    - Outputs: Sent email list for context.  
    - Edge cases: No sent emails found, query errors.  
  - **Structured Output Parser1**  
    - Type: LangChain output parser (structured)  
    - Role: Parses AI output into JSON with label and label ID fields, auto-fixes minor JSON issues.  
    - Config: JSON schema example provided to enforce output format.  
    - Inputs: AI Agent3 output.  
    - Outputs: Parsed JSON with label data.  
    - Edge cases: Parsing failure, invalid AI output.  
  - **Sticky Note3**  
    - Documentation node explaining AI classification logic.

#### 1.3 Label Management and Application

- **Overview:** Ensures the Gmail label suggested by AI exists (creates if missing), fetches all existing labels, filters the correct label ID, and applies it to the email thread to organize inbox.  
- **Nodes Involved:**  
  - Create Label if Doesn't exist1  
  - Get Existing Labels1  
  - Filter1  
  - Loop Over Items1  
  - Add label to thread1  
  - Sticky Note4

- **Node Details:**  
  - **Create Label if Doesn't exist1**  
    - Type: Gmail node (label create)  
    - Role: Creates the label suggested by AI if it does not exist.  
    - Config: Label name from AI output ('output.label'), continues workflow on error if label exists.  
    - Inputs: AI classification parsed label name.  
    - Outputs: Creation result or error continuation.  
    - Edge cases: Label creation failure, permission issues.  
  - **Get Existing Labels1**  
    - Type: Gmail node (label get all)  
    - Role: Retrieves all current Gmail labels for filtering.  
    - Config: Returns all labels without pagination.  
    - Inputs: Output of label creation node.  
    - Outputs: List of labels.  
  - **Filter1**  
    - Type: Filter node  
    - Role: Filters labels to find one matching the AI suggested label name.  
    - Config: Condition equals label name to AI output label.  
    - Inputs: Output of Get Existing Labels1 and Loop Over Items1.  
    - Outputs: The matched label.  
  - **Loop Over Items1**  
    - Type: SplitInBatches node  
    - Role: Iterates over label list for processing or batch operations.  
    - Config: Default batch size.  
    - Inputs: Output of AI Agent3 and Add label node.  
    - Outputs: Batches to process label application.  
  - **Add label to thread1**  
    - Type: Gmail node (thread label add)  
    - Role: Applies the filtered label ID to the email thread.  
    - Config: Uses thread ID from email, label ID from filter.  
    - Inputs: Filter1 output.  
    - Outputs: Label application result.  
  - **Sticky Note4**  
    - Documentation on label management logic and error tolerance.

#### 1.4 AI-Based Urgency Detection and Summarization

- **Overview:** Separately analyzes the email to classify urgency as "urgent" or "normal" and generates a short summary for notifications.  
- **Nodes Involved:**  
  - AI Agent1  
  - OpenRouter Chat Model3  
  - Google Gemini Chat Model3  
  - Code1 (JSON parsing)  
  - If1 (urgency check)  
  - Wait1  
  - Simple Memory1  
  - Sticky Note5

- **Node Details:**  
  - **AI Agent1**  
    - Type: LangChain AI agent  
    - Role: Evaluates email urgency and summarizes content in JSON format.  
    - Config: Prompt instructs classification into "urgent" or "normal" and summary generation.  
    - Inputs: Email content from Get a message1 node.  
    - Outputs: JSON with "urgency" and "summary" fields.  
    - Edge cases: AI output parsing errors, ambiguous urgency.  
  - **OpenRouter Chat Model3 & Google Gemini Chat Model3**  
    - Type: Language model nodes  
    - Role: Provide fallback and complementary language model inference for urgency analysis.  
    - Config: OpenRouter model "deepseek-chat-v3-0324:free".  
  - **Code1**  
    - Type: Code node (JavaScript)  
    - Role: Cleans and parses AI JSON output safely, extracts urgency and summary fields.  
    - Config: Strips Markdown code blocks, attempts JSON parse, falls back with error.  
    - Inputs: AI Agent1 output.  
    - Outputs: Clean JSON with urgency and summary.  
    - Edge cases: Malformed AI output, parse failure.  
  - **If1**  
    - Type: If node  
    - Role: Checks if urgency equals "urgent".  
    - Inputs: Output of Code1.  
    - Outputs: Routes urgent emails to notification nodes, normal emails to wait node.  
  - **Wait1**  
    - Type: Wait node  
    - Role: Adds a 15-second delay before further processing (for non-urgent emails).  
  - **Simple Memory1**  
    - Type: LangChain memory buffer window  
    - Role: Stores session-based memory keyed on label ID from Get Existing Labels1.  
  - **Sticky Note5**  
    - Documentation on urgency detection and notification trigger.

#### 1.5 Urgent Notification Dispatch

- **Overview:** Sends alert messages with email summary and snippet to Telegram chat and WhatsApp if email urgency is "urgent".  
- **Nodes Involved:**  
  - Send a text message1 (Telegram)  
  - Send message1 (WhatsApp)  

- **Node Details:**  
  - **Send a text message1**  
    - Type: Telegram node  
    - Role: Sends a text message to a configured Telegram chat ID.  
    - Config: Message body includes recipient, summary, and snippet from email. Disabled attribution. Retry on failure enabled.  
    - Inputs: Routed from If1 on urgent condition.  
    - Outputs: Telegram delivery status.  
    - Edge cases: Chat ID invalid, Telegram API errors, message formatting issues.  
  - **Send message1**  
    - Type: WhatsApp node  
    - Role: Sends WhatsApp message to specified phone number ID and recipient number.  
    - Config: Message includes recipient, summary, snippet; preview URL disabled.  
    - Inputs: Routed from If1 on urgent condition.  
    - Outputs: WhatsApp delivery status.  
    - Edge cases: Phone number invalid, WhatsApp API errors, quota limits.

---

### 3. Summary Table

| Node Name                    | Node Type                                   | Functional Role                            | Input Node(s)           | Output Node(s)                     | Sticky Note                                      |
|------------------------------|---------------------------------------------|-------------------------------------------|-------------------------|----------------------------------|-------------------------------------------------|
| Gmail Trigger1               | Gmail Trigger                               | Detects new incoming emails                | None                    | Get a message1                   | ## Trigger: New Email Received                   |
| Get a message1               | Gmail                                       | Fetches full email content                  | Gmail Trigger1          | AI Agent3, AI Agent1             | ## Fetch Full Email Content                       |
| AI Agent3                   | LangChain AI Agent                          | Classifies email with domain-specific label | Get a message1, Check Sent2, Check Sent3 | Loop Over Items1                  | ## AI-Powered Categorization                      |
| OpenRouter Chat Model2       | LangChain Chat Model                        | Language model for classification           | AI Agent3               | AI Agent3                       |                                                 |
| Google Gemini Chat Model2    | LangChain Chat Model                        | Fallback language model for classification  | AI Agent3               | AI Agent3                       |                                                 |
| Check Sent2                 | GmailTool                                   | Checks if email was sent to sender          | AI Agent3               | AI Agent3                       |                                                 |
| Check Sent3                 | GmailTool                                   | (Unconnected in JSON, likely fallback)      |                         | AI Agent3                       |                                                 |
| Structured Output Parser1    | LangChain Output Parser                     | Parses AI classification output to JSON    | OpenAI Chat Model       | AI Agent3                       |                                                 |
| Create Label if Doesn't exist1 | Gmail Label Create                          | Creates label if not existing                | Loop Over Items1         | Get Existing Labels1             | ## Label Management and Application               |
| Get Existing Labels1         | Gmail Label Retrieve                        | Retrieves all Gmail labels                   | Create Label if Doesn't exist1 | Filter1                      |                                                 |
| Filter1                     | Filter                                      | Matches AI label name with existing labels  | Get Existing Labels1     | Add label to thread1             |                                                 |
| Loop Over Items1             | SplitInBatches                              | Iterates over label items for processing    | AI Agent3, Add label to thread1 | Create Label if Doesn't exist1, AI Agent1 |                                                 |
| Add label to thread1         | Gmail Thread Label Add                      | Applies label to email thread                | Filter1                 | Loop Over Items1                 |                                                 |
| AI Agent1                   | LangChain AI Agent                          | Analyzes email urgency and summarizes       | Get a message1          | Code1                          | ## Urgency Analysis and Summarization             |
| OpenRouter Chat Model3       | LangChain Chat Model                        | Language model for urgency analysis          | AI Agent1               | AI Agent1                      |                                                 |
| Google Gemini Chat Model3    | LangChain Chat Model                        | Fallback language model for urgency analysis | AI Agent1               | AI Agent1                      |                                                 |
| Code1                       | Code                                        | Parses AI JSON output for urgency and summary | AI Agent1               | If1                           |                                                 |
| If1                         | If                                          | Checks if email urgency is "urgent"          | Code1                   | Send a text message1, Send message1, Wait1 |                                                 |
| Wait1                       | Wait                                        | Delays further processing for non-urgent emails | If1                     | AI Agent1                      |                                                 |
| Simple Memory1               | LangChain Memory Buffer                     | Stores session memory keyed on label ID     | Get Existing Labels1     | AI Agent1                      |                                                 |
| Send a text message1         | Telegram                                    | Sends urgent email alert to Telegram chat   | If1                     |                              |                                                 |
| Send message1                | WhatsApp                                    | Sends urgent email alert to WhatsApp number | If1                     |                              |                                                 |
| Sticky Note1                | Sticky Note                                 | Documentation: Trigger block                  | None                    | None                         | ## Trigger: New Email Received                   |
| Sticky Note2                | Sticky Note                                 | Documentation: Fetch full email content       | None                    | None                         | ## Fetch Full Email Content                       |
| Sticky Note3                | Sticky Note                                 | Documentation: AI classification logic       | None                    | None                         | ## AI-Powered Categorization                      |
| Sticky Note4                | Sticky Note                                 | Documentation: Label management and application | None                    | None                         | ## Label Management and Application               |
| Sticky Note5                | Sticky Note                                 | Documentation: Urgency analysis and notification | None                    | None                         | ## Urgency Analysis and Summarization             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger node ("Gmail Trigger1")**  
   - Type: Gmail Trigger  
   - Configure to poll every minute (pollTimes: everyMinute)  
   - No email filters applied  
   - Connect output to next node.

2. **Add Gmail node ("Get a message1")**  
   - Type: Gmail (get message)  
   - Operation: Get message  
   - Use message ID from trigger node: `{{$json["id"]}}`  
   - Connect input to Gmail Trigger1 output.

3. **Add LangChain AI Agent node ("AI Agent3") for classification**  
   - Type: LangChain agent  
   - Text prompt includes detailed email metadata and classification rules (see section 2.2 for prompt content)  
   - Use multi-model fallback: OpenRouter Chat Model2 and Google Gemini Chat Model2  
   - Connect input from Get a message1 and from GmailTool nodes (Check Sent2 and Check Sent3 for context).

4. **Add GmailTool node ("Check Sent2")**  
   - Type: GmailTool (getAll)  
   - Operation: Search emails sent to the address extracted from AI Agent3 context `{{ $fromAI('email') }}`  
   - Label filter: SENT  
   - Connect input from AI Agent3.

5. **Add GmailTool node ("Check Sent3")**  
   - (Optional fallback or additional check, connect to AI Agent3 if needed).

6. **Add LangChain Output Parser node ("Structured Output Parser1")**  
   - Type: Output Parser Structured  
   - Auto-fix enabled  
   - Schema example: {"label": "Label name", "label ID": "label ID"}  
   - Connect output from AI Agent3.

7. **Add SplitInBatches node ("Loop Over Items1")**  
   - Type: SplitInBatches  
   - Default batch size  
   - Connect output from AI Agent3.

8. **Add Gmail node ("Create Label if Doesn't exist1")**  
   - Type: Gmail (label create)  
   - Operation: Create label  
   - Label name from parsed AI output: `={{ $json.output.label }}`  
   - On error: Continue workflow (for label already exists)  
   - Connect input from Loop Over Items1 output.

9. **Add Gmail node ("Get Existing Labels1")**  
   - Type: Gmail (label get all)  
   - Return all labels  
   - Connect input from Create Label if Doesn't exist1.

10. **Add Filter node ("Filter1")**  
    - Condition: label name equals AI suggested label name (`={{ $('Loop Over Items1').item.json.output.label }}`)  
    - Connect input from Get Existing Labels1.

11. **Add Gmail node ("Add label to thread1")**  
    - Type: Gmail (add labels to thread)  
    - Thread ID from Get a message1: `={{ $('Get a message1').item.json.threadId }}`  
    - Label IDs from Filter1 output: `={{ $json.id }}`  
    - Connect input from Filter1.

12. **Add LangChain AI Agent node ("AI Agent1") for urgency and summary**  
    - Type: LangChain agent  
    - Prompt: Classify email urgency ("urgent" or "normal") and generate a short summary (max 2 sentences)  
    - Use OpenRouter Chat Model3 and Google Gemini Chat Model3 as language models  
    - Connect input from Get a message1.

13. **Add Code node ("Code1")**  
    - Type: Code  
    - JavaScript to clean and parse AI JSON output for fields: urgency and summary  
    - Connect input from AI Agent1.

14. **Add If node ("If1")**  
    - Condition: urgency equals "urgent"  
    - Connect input from Code1.

15. **Add Telegram node ("Send a text message1")**  
    - Type: Telegram  
    - Chat ID: your configured chat ID  
    - Text body includes To address, summary, and email snippet  
    - Connect input from If1 "true" branch.

16. **Add WhatsApp node
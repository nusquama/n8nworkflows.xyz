Auto-Reply & Create Linear Tickets from Gmail with GPT-5, gotoHuman & Human Review

https://n8nworkflows.xyz/workflows/auto-reply---create-linear-tickets-from-gmail-with-gpt-5--gotohuman---human-review-10065


# Auto-Reply & Create Linear Tickets from Gmail with GPT-5, gotoHuman & Human Review

### 1. Workflow Overview

This n8n workflow automates the processing of incoming emails from Gmail by leveraging advanced AI models (GPT-5) and a human-in-the-loop review system (gotoHuman). Its primary purpose is to classify customer support emails, draft personalized replies, route them for human review, and create tickets in Linear for bug reports or feature requests. The workflow continuously improves through self-learning by integrating approved past examples into AI prompts.

Logical blocks in the workflow:

- **1.1 Input Reception:** Trigger on new Gmail emails, batch processing of emails (including test data).
- **1.2 Email Classification:** AI-based classification of emails into predefined categories, enhanced by fetching approved classification examples.
- **1.3 Email Reply Drafting:** AI drafts a reply based on the email content and previously approved replies for the classified category.
- **1.4 Human Review:** The drafted reply and classification are sent for human review via gotoHuman, allowing edits, approval, or retry.
- **1.5 Reply Sending:** Upon human approval, the reply is sent back as a Gmail thread reply.
- **1.6 Ticket Creation:** When the email is classified as a bug report or feature request, an AI agent drafts a Linear ticket, which is then created.
- **1.7 Self-learning Integration:** The workflow fetches previously approved classification and reply examples from gotoHuman to improve AI performance over time.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block triggers the workflow on new incoming emails from Gmail and supports test runs with sample emails for development or demonstration purposes.

**Nodes Involved:**
- New Email
- Email to process
- Test Emails (to fill memory)
- Test Emails 2nd batch
- When clicking ‘Execute workflow’
- Is test run?

**Node Details:**

- **New Email**  
  - Type: Gmail Trigger  
  - Role: Watches the Gmail inbox for new emails every minute.  
  - Configurations: Polls every minute, retrieves full email data (not simplified).  
  - Inputs: None (trigger node).  
  - Outputs: Passes new email data downstream.  
  - Edge cases: Gmail API rate limits, authentication errors.  
  - Credentials: Gmail OAuth2 required.

- **Email to process**  
  - Type: SplitInBatches  
  - Role: Splits incoming emails into batches for sequential processing.  
  - Inputs: New Email or test emails.  
  - Outputs: Single email per batch to classifier and other downstream nodes.  
  - Edge cases: Empty input, batch size defaults.

- **Test Emails (to fill memory)**  
  - Type: Code (JavaScript)  
  - Role: Provides a sample dataset of emails for testing or training the AI models.  
  - Inputs: Manual trigger node or workflow execution.  
  - Outputs: Array of sample email JSON objects.  
  - Edge cases: Static dataset; no runtime variability.

- **Test Emails 2nd batch**  
  - Type: Code (JavaScript)  
  - Role: Provides additional sample emails for test or memory filling.  
  - Inputs: Manual trigger.  
  - Outputs: Sample emails JSON array.  
  - Edge cases: Same as above.

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Allows manual execution to test workflow with test emails.  
  - Inputs: None.  
  - Outputs: Triggers Test Emails 2nd batch node.  
  - Edge cases: None.

- **Is test run?**  
  - Type: Switch  
  - Role: Determines if the current run is a real email or a test execution.  
  - Inputs: Human response node outputs.  
  - Outputs: Routes to 'Reply to thread' node if real, or to 'Is bug of feature request?' for further processing.  
  - Edge cases: Incorrect flag handling might misroute.  
  - Conditions: Checks if the New Email node is executed to distinguish real vs. test.

---

#### 1.2 Email Classification

**Overview:**  
This block classifies incoming emails into categories such as bug report, feature request, sales opportunity, etc., using an AI agent enhanced with previously approved classification examples fetched from gotoHuman.

**Nodes Involved:**
- AI Classifier
- OpenAI Chat Model
- Structured Output1
- Fetch approved classification examples
- Email to process (input for fetching examples)

**Node Details:**

- **AI Classifier**  
  - Type: Langchain AI Agent  
  - Role: Classifies email contents into predefined categories with importance flag.  
  - Configurations: Uses prompt with system message directing classification.  
  - Inputs: Email text, subject, to/from fields from 'Email to process'.  
  - Outputs: AI classification result JSON object.  
  - Key Expressions: Uses email fields dynamically in prompt.  
  - Edge cases: AI misclassification, network/API errors.  
  - Version: Langchain Agent v2.2.

- **OpenAI Chat Model**  
  - Type: Langchain model  
  - Role: Language model supporting AI Classifier node.  
  - Configurations: Model set to "gpt-5-nano".  
  - Edge cases: API limits, model availability.

- **Structured Output1**  
  - Type: Langchain Structured Output Parser  
  - Role: Parses AI Classifier's raw output into structured JSON with classification and importance boolean.  
  - Configuration: Manual JSON schema specifying categories and importance.  
  - Edge cases: Parsing failure if AI returns unexpected format.

- **Fetch approved classification examples**  
  - Type: HTTP Request  
  - Role: Queries gotoHuman API for previously approved classification examples to provide personalized training data to AI.  
  - Configuration: Sends formId (to be set by user), requests meta.emailText and classification fields.  
  - Inputs: Email batch.  
  - Outputs: List of approved classification examples.  
  - Edge cases: API auth failure, empty results, missing formId.  
  - Credentials: gotoHuman API key required.

---

#### 1.3 Email Reply Drafting

**Overview:**  
This block uses AI to draft a reply to the classified email, incorporating previously approved replies for the email’s category to personalize tone and style.

**Nodes Involved:**
- AI Email Writer
- OpenAI Chat Model2
- Structured Output
- Fetch approved email examples
- Set Prompt
- Set (edited) prompt

**Node Details:**

- **AI Email Writer**  
  - Type: Langchain AI Agent  
  - Role: Generates a draft email reply based on the received email and approved past replies.  
  - Configuration: Prompt includes user email and optionally previous approved replies for the email classification.  
  - Inputs: Email content, prompt text (from Set Prompt or edited prompt).  
  - Outputs: Draft reply text.  
  - Edge cases: AI hallucination, incomplete drafts, API failures.  
  - Version: Langchain Agent v2.2.

- **OpenAI Chat Model2**  
  - Type: Langchain model  
  - Role: Supports AI Email Writer node with GPT-5-nano model.  
  - Edge cases: Same as OpenAI Chat Model.

- **Structured Output**  
  - Type: Langchain Structured Output Parser  
  - Role: Parses AI Email Writer output into structured JSON with at least "textEmailDraft".  
  - Edge cases: Parsing mismatch.

- **Fetch approved email examples**  
  - Type: HTTP Request  
  - Role: Fetches approved email reply texts from gotoHuman filtered by classification to improve AI email drafting.  
  - Configuration: Uses formId, requests only 'emailReply' field, filters by classification.  
  - Edge cases: API errors, empty data, missing formId.

- **Set Prompt**  
  - Type: Set  
  - Role: Constructs the system prompt for the AI Email Writer including references to previous approved email replies.  
  - Configuration: Builds prompt string dynamically with fallback if no prior replies exist.  
  - Edge cases: Empty or missing prior replies.

- **Set (edited) prompt**  
  - Type: Set  
  - Role: Updates prompt dynamically after human edits or retries, passing the latest prompt for AI re-processing.  
  - Inputs: Human review node output (edited prompt, review ID).  
  - Outputs: Updated prompt for AI Email Writer.  
  - Edge cases: Missing or malformed edited prompt.

---

#### 1.4 Human Review

**Overview:**  
This block routes the AI-generated classification and email draft to a human reviewer through the gotoHuman platform for approval, editing, or retry.

**Nodes Involved:**
- Human review (gotoHuman node)
- Human response (Switch)
- Set (edited) prompt

**Node Details:**

- **Human review**  
  - Type: gotoHuman node  
  - Role: Sends AI classification and draft reply to human reviewers for validation and editing.  
  - Configuration: Maps email content, classification, and draft reply fields; uses a specific review template ID (to be set by user). Supports updating existing reviews.  
  - Inputs: Draft reply, classification, email content.  
  - Outputs: Human responses including approval status, edits, or retry requests.  
  - Edge cases: gotoHuman API errors, template ID missing or invalid.  
  - Credentials: gotoHuman API key.

- **Human response**  
  - Type: Switch  
  - Role: Routes workflow based on human reviewer’s decision: Approved, Rejected, or Retry.  
  - Configuration: Checks the ‘response’ field for exact values "approved", "rejected", or retries based on type.  
  - Outputs:  
    - Approved: proceeds to send reply.  
    - Rejected: no action (email processed).  
    - Retry: loops back to AI Email Writer with updated prompt.  
  - Edge cases: Unexpected human responses, missing fields.

- **Set (edited) prompt**  
  - See above in 1.3; also used here to update prompt after human edits.

---

#### 1.5 Reply Sending

**Overview:**  
Sends the human-approved reply as a response to the original email thread in Gmail.

**Nodes Involved:**
- Reply to thread
- Is test run?
- No Action
- Email Processed

**Node Details:**

- **Reply to thread**  
  - Type: Gmail node  
  - Role: Sends the approved email draft as a reply to the original email thread.  
  - Configuration: Uses message ID from original email to reply in thread, disables attribution footer, sends plain text email.  
  - Inputs: Approved email reply text, original email metadata.  
  - Outputs: Continues to check if ticket creation is needed.  
  - Edge cases: Gmail API errors, invalid message ID, authentication errors.  
  - Credentials: Gmail OAuth2 required.

- **Is test run?**  
  - See above; routes test runs differently (direct to sending or ticket creation).

- **No Action**  
  - Type: No Operation  
  - Role: Ends flow for rejected replies or no further action needed.

- **Email Processed**  
  - Type: No Operation  
  - Role: Marks completion of processing an email.

---

#### 1.6 Ticket Creation

**Overview:**  
Automatically creates a ticket in Linear for emails classified as bug reports or feature requests, based on AI-generated ticket details.

**Nodes Involved:**
- Is bug of feature request? (filter)
- AI Agent (ticket creation)
- GPT-5-mini
- Structured Output Parser
- Create an issue
- Email Processed

**Node Details:**

- **Is bug of feature request?**  
  - Type: Filter  
  - Role: Checks if classification is either "bug_report" or "feature_request".  
  - Inputs: Human review node outputs classification value.  
  - Outputs:  
    - True: proceeds to AI Agent to generate ticket details.  
    - False: ends without creating ticket.  
  - Edge cases: Misclassification, empty classification.

- **AI Agent (ticket creation)**  
  - Type: Langchain AI Agent  
  - Role: Creates structured ticket details (title, description, priority) for Linear based on email content and classification.  
  - Configuration: Prompt instructs AI to generate ticket info for product team.  
  - Inputs: Email subject and body, classification.  
  - Outputs: Unstructured AI output.  
  - Edge cases: AI misunderstanding, irrelevant ticket info.

- **GPT-5-mini**  
  - Type: Langchain language model  
  - Role: Provides GPT-5-nano model for AI Agent.  
  - Edge cases: API limits.

- **Structured Output Parser**  
  - Type: Langchain Structured Output Parser  
  - Role: Parses AI Agent output into JSON with required fields: title, description, priority (1-4).  
  - Edge cases: Parsing failure.

- **Create an issue**  
  - Type: Linear node  
  - Role: Creates a new issue in Linear using parsed ticket data.  
  - Configurations: Uses a predefined teamId, priority, description, and title from AI output.  
  - Inputs: Parsed ticket data.  
  - Outputs: Marks email as processed after ticket creation.  
  - Edge cases: Linear API errors, invalid teamId, authentication failure.  
  - Credentials: Linear API key.

- **Email Processed**  
  - See above; marks completion.

---

#### 1.7 Self-learning Integration via gotoHuman

**Overview:**  
The workflow integrates with gotoHuman to fetch previously approved classification and email reply examples for training and refining the AI agents continuously.

**Nodes Involved:**
- Fetch approved classification examples
- Fetch approved email examples

**Node Details:**

- **Fetch approved classification examples**  
  - See above in 1.2; provides training data for classifier.

- **Fetch approved email examples**  
  - Type: HTTP Request  
  - Role: Fetches approved email reply examples grouped by classification from gotoHuman.  
  - Configuration: Filters responses by classification matching current email classification.  
  - Inputs: Email to process, classification output.  
  - Outputs: Previous approved email replies for Set Prompt node.  
  - Edge cases: API errors, missing or empty data.

---

### 3. Summary Table

| Node Name                         | Node Type                            | Functional Role                             | Input Node(s)                          | Output Node(s)                          | Sticky Note                                                                                              |
|----------------------------------|------------------------------------|---------------------------------------------|--------------------------------------|----------------------------------------|--------------------------------------------------------------------------------------------------------|
| New Email                        | Gmail Trigger                      | Trigger on new incoming emails               | -                                    | Email to process                       | ## Self-learning Email Classifier: AI classifies received email using gotoHuman examples               |
| Email to process                 | SplitInBatches                    | Batch processing of emails                    | New Email, Test Emails, Test Emails 2nd batch | Fetch approved classification examples, AI Classifier |                                                                                                        |
| Test Emails (to fill memory)     | Code                             | Provides sample emails for testing            | Manual trigger                       | Email to process                       |                                                                                                        |
| Test Emails 2nd batch            | Code                             | Additional sample emails for testing          | Manual trigger                       | Email to process                       |                                                                                                        |
| When clicking ‘Execute workflow’ | Manual Trigger                   | Manual workflow execution trigger             | -                                    | Test Emails 2nd batch                 |                                                                                                        |
| Is test run?                    | Switch                           | Routes test vs. real email processing         | Human response                      | Reply to thread, Is bug of feature request |                                                                                                        |
| AI Classifier                   | Langchain AI Agent               | Classifies email into categories              | Fetch approved classification examples, Email to process | Fetch approved email examples         |                                                                                                        |
| OpenAI Chat Model               | Langchain LM                    | Supports AI Classifier node                    | -                                    | AI Classifier                        |                                                                                                        |
| Structured Output1              | Langchain Output Parser         | Parses classification output                   | AI Classifier                      | Fetch approved email examples         |                                                                                                        |
| Fetch approved classification examples | HTTP Request                  | Fetches approved classification examples from gotoHuman | Email to process                   | AI Classifier                        |                                                                                                        |
| Fetch approved email examples    | HTTP Request                    | Fetches approved email replies filtered by classification | AI Classifier                    | Set Prompt                          |                                                                                                        |
| Set Prompt                     | Set                              | Creates prompt for AI Email Writer             | Fetch approved email examples        | AI Email Writer                    | ## Self-learning Email Writer: AI drafts replies using previous approved examples                      |
| AI Email Writer                | Langchain AI Agent               | Drafts email reply based on email and prompt  | Set Prompt, Set (edited) prompt     | Human review                      |                                                                                                        |
| OpenAI Chat Model2             | Langchain LM                    | Supports AI Email Writer node                   | -                                    | AI Email Writer                    |                                                                                                        |
| Structured Output              | Langchain Output Parser         | Parses AI Email Writer's reply draft           | AI Email Writer                    | Human review                      |                                                                                                        |
| Human review                   | gotoHuman                       | Sends classification and draft reply for human review | AI Email Writer                   | Human response                   | ## Human Review: Review and edit AI outputs in gotoHuman, retry or approve                             |
| Human response                 | Switch                         | Routes based on human approval decision        | Human review                      | Reply to thread, No Action, Set (edited) prompt |                                                                                                        |
| Set (edited) prompt            | Set                            | Updates AI prompt after human edits or retry  | Human response                   | AI Email Writer                    |                                                                                                        |
| Reply to thread               | Gmail node                     | Sends approved reply as Gmail thread reply     | Human response                   | Is bug of feature request           | ## Send approved Reply: Sends the approved response as reply to email thread                           |
| Is bug of feature request?    | Filter                         | Checks if classification is bug or feature request | Reply to thread, Is test run?        | AI Agent (ticket creation), No Action | ## Create ticket: Creates a ticket for bugs or feature requests                                         |
| AI Agent (ticket creation)    | Langchain AI Agent             | Generates ticket title, description, priority  | Is bug of feature request?          | Structured Output Parser            |                                                                                                        |
| GPT-5-mini                    | Langchain LM                  | Supports ticket creation AI agent               | -                                    | AI Agent (ticket creation)           |                                                                                                        |
| Structured Output Parser      | Langchain Output Parser       | Parses ticket creation AI output                 | AI Agent (ticket creation)          | Create an issue                   |                                                                                                        |
| Create an issue               | Linear node                  | Creates a ticket in Linear                        | Structured Output Parser            | Email Processed                   |                                                                                                        |
| Email Processed               | No Operation                 | Marks completion of email processing             | Create an issue, No Action           | Email to process                   |                                                                                                        |
| No Action                    | No Operation                 | Ends flow for rejected emails                     | Human response                   | Email Processed                   |                                                                                                        |
| Sticky Notes (multiple nodes) | Sticky Note                  | Provide documentation, explanations, and branding | Various                          | Various                          | Refer to sticky note contents in the workflow for detailed explanations and images                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger node**  
   - Type: Gmail Trigger  
   - Poll interval: every minute  
   - Retrieve full email content (not simplified)  
   - Configure Gmail OAuth2 credentials.

2. **Add SplitInBatches node ("Email to process")**  
   - Connect from Gmail Trigger output.  
   - Use default batch size to process one email at a time.

3. **Add HTTP Request node ("Fetch approved classification examples")**  
   - Connect from "Email to process" (second output).  
   - URL: `https://api.gotohuman.com/fetchResponses`  
   - Query parameters:  
     - `formId` (set your gotoHuman review template ID)  
     - `fieldIds`: `meta.emailText,classification`  
     - `approvedValuesOnly`: `true`  
   - Authentication: Use predefined gotoHuman API credentials.

4. **Add Langchain OpenAI Chat Model node ("OpenAI Chat Model")**  
   - Model: "gpt-5-nano"  
   - No additional options needed.

5. **Add Langchain AI Agent node ("AI Classifier")**  
   - Connect AI Classifier to OpenAI Chat Model (ai_languageModel).  
   - Input text uses dynamic expressions with email fields from "Email to process".  
   - System message prompt instructs classification into categories: product question, bug report, feature request, account & billing, sales opportunity, other.  
   - Use examples fetched from "Fetch approved classification examples" to improve AI.  
   - Enable output parser with schema for classification and importance.

6. **Add Langchain Structured Output Parser node ("Structured Output1")**  
   - Connect from AI Classifier's ai_outputParser.  
   - Define JSON schema with properties: classifiedAs (enum categories), important (boolean).

7. **Add HTTP Request node ("Fetch approved email examples")**  
   - Connect from AI Classifier output.  
   - Query gotoHuman for approved email replies filtered by classification from classifier.  
   - Use same authentication as previous gotoHuman HTTP node.

8. **Add Set node ("Set Prompt")**  
   - Connect from "Fetch approved email examples".  
   - Compose prompt string for AI Email Writer including previous replies if available.

9. **Add Langchain OpenAI Chat Model node ("OpenAI Chat Model2")**  
   - Model: "gpt-5-nano".

10. **Add Langchain AI Agent node ("AI Email Writer")**  
    - Connect main output from "Set Prompt" to AI Email Writer.  
    - Attach ai_languageModel input from "OpenAI Chat Model2".  
    - Use prompt from Set Prompt node dynamically.  
    - Enable structured output parser for reply draft.

11. **Add Langchain Structured Output Parser node ("Structured Output")**  
    - Connect from AI Email Writer ai_outputParser.  
    - Schema expects object with "textEmailDraft" string.

12. **Add gotoHuman node ("Human review")**  
    - Connect from AI Email Writer main output.  
    - Map fields: email content, sender, classification, drafted reply.  
    - Use your gotoHuman review template ID in configuration.  
    - Allow updating existing review using review ID from edited prompt node.

13. **Add Switch node ("Human response")**  
    - Connect from Human review.  
    - Add rules for:  
      - Approved: response equals "approved"  
      - Rejected: response equals "rejected"  
      - Retry: type equals "chat" (for retries)  
    - Route accordingly.

14. **Add Set node ("Set (edited) prompt")**  
    - Connect from "Human response" Retry output.  
    - Update prompt and reviewToUpdate fields based on edits made in gotoHuman.

15. **Connect "Set (edited) prompt" back to AI Email Writer** to enable retry with updated prompt.

16. **Add Gmail node ("Reply to thread")**  
    - Connect from "Human response" Approved output.  
    - Configure to reply to original Gmail message ID from "New Email".  
    - Email Type: text, disable attribution.

17. **Add Switch node ("Is test run?")**  
    - Connect from "Human response".  
    - Route based on whether "New Email" node was executed (real email) or not (test run).  
    - Approved replies go to "Reply to thread" for real emails; test runs skip sending.

18. **Add Filter node ("Is bug of feature request?")**  
    - Connect from "Reply to thread" and from "Is test run?" test run output.  
    - Condition: classification is "bug_report" or "feature_request".

19. **Add Langchain OpenAI Chat Model node ("GPT-5-mini")**  
    - Model: "gpt-5-nano".

20. **Add Langchain AI Agent node ("AI Agent")**  
    - Connect from Filter node true output.  
    - Input prompt requests creation of ticket title, description, priority based on email content.  
    - Connect ai_languageModel input from "GPT-5-mini".

21. **Add Langchain Structured Output Parser node ("Structured Output Parser")**  
    - Connect from AI Agent ai_outputParser.  
    - Schema expects title (string), description (string), priority (enum 1-4).

22. **Add Linear node ("Create an issue")**  
    - Connect from Structured Output Parser.  
    - Configure with your Linear teamId and credentials.  
    - Use output fields to populate issue title, description, priority.

23. **Add No Operation nodes ("No Action" and "Email Processed")**  
    - Connect "Rejected" output of Human response to "No Action".  
    - Connect final nodes to "Email Processed" to mark completion.

24. **Add Sticky Notes** at appropriate places for documentation.

25. **Set all credentials:**  
    - Gmail OAuth2 for email nodes.  
    - OpenAI for Langchain nodes.  
    - gotoHuman API key for HTTP and gotoHuman nodes.  
    - Linear API key for ticket creation.

26. **Set formId and review template ID in gotoHuman-related nodes** to your actual template.

27. **Test workflow with sample emails** using manual trigger and test email code nodes.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Context or Link                                                                                                                                                |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------|
| The workflow integrates gotoHuman for human-in-the-loop review and continuous AI learning from approved classification and reply examples. To set up gotoHuman, create or import the review template "Support email agent" with ID `6fzuCJlFYJtlu9mGYcVT` and use that template ID in the workflow nodes.                                                                                                                                                                                                                                                                                                                                                                                                                                                       | gotoHuman platform: https://gotohuman.com/                                                                                                                    |
| Branding images used in sticky notes show gotoHuman logos and review screen for visual reference.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | ![gotoHuman logo](https://cdn1.gotohuman.com/public%2Fn8n-templates%2FgotoHuman%20full%20logo%201224px.png?alt=media) and ![review screen](https://cdn1.gotohuman.com/public%2Fn8n-templates%2Femail-to-ticket%2Fgth-review-screen.jpg?alt=media) |
| The AI agents use GPT-5-nano model via Langchain nodes for classification, drafting, and ticket creation to ensure consistent AI performance across tasks.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | OpenAI GPT-5 info (internal use).                                                                                                                             |
| The workflow includes example test emails covering diverse categories (product questions, bugs, feature requests, sales, billing, others) to simulate real-world inputs and to help train the AI models.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | See "Test Emails (to fill memory)" and "Test Emails 2nd batch" nodes.                                                                                        |
| Linear is used for ticket creation with a preset team ID; customize it for your own team and project in Linear.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Linear API docs: https://developers.linear.app/docs/graphql/getting-started                                                                                   |
| Important: Before importing this workflow template into your n8n, ensure you have the gotoHuman node installed and set up with valid credentials as it is a prerequisite.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Workflow notes                                                                                                                                                |
| Customize the classification categories and AI prompts to match your company's taxonomy and tone preferences. Filter training data by reviewer if needed for personalized AI style.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Workflow customization suggestions                                                                                                                          |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All processed data are legal and public.
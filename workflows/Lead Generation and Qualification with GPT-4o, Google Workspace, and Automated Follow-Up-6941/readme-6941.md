Lead Generation and Qualification with GPT-4o, Google Workspace, and Automated Follow-Up

https://n8nworkflows.xyz/workflows/lead-generation-and-qualification-with-gpt-4o--google-workspace--and-automated-follow-up-6941


# Lead Generation and Qualification with GPT-4o, Google Workspace, and Automated Follow-Up

---

## 1. Workflow Overview

This workflow automates the process of lead generation, enrichment, classification, and follow-up using GPT-4o AI models combined with Google Workspace services (Sheets, Gmail, Calendar) within n8n. It targets B2B sales teams aiming to efficiently qualify incoming leads, enrich lead data with external insights, classify leads into categories (demo-ready, nurture, drop), and trigger tailored email and calendar follow-ups accordingly.

The workflow’s logic is structured into these main functional blocks:

- **1.1 Input Reception and Logging**: Captures lead submissions via a form trigger and logs raw data into Google Sheets.
- **1.2 AI Initial Response Agent**: Uses AI to determine if the lead inquiry can be answered automatically or should be escalated.
- **1.3 AI Lead Enrichment**: Enriches lead data with company info, pain points, and relevant metadata via AI-assisted web search.
- **1.4 Defining ICP and Lead Criteria**: Holds configurable Ideal Customer Profile (ICP) and lead qualification criteria.
- **1.5 AI Lead Classification**: Classifies enriched leads into “demo-ready”, “nurture”, or “drop” categories using GPT-4o.
- **1.6 Lead Follow-Up Sequences**: For each classification, triggers appropriate email sequences and calendar scheduling:
  - Demo-ready: Immediate calendar invite and update.
  - Nurture: Educational email series followed by demo scheduling.
  - Drop: Polite rejection email.
- **1.7 Google Sheets Updates**: Updates lead records with enrichment and classification data.

Sticky notes throughout the workflow provide configuration guidance, future improvement ideas, and best practices for each block.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception and Logging

- **Overview:** Captures incoming leads from a web form and logs raw submission data into a Google Sheet for tracking.

- **Nodes Involved:** 
  - Input Form
  - Log Lead to Sheet
  - Get all columns
  - Return latest column

- **Node Details:**

  - **Input Form**
    - Type: Form Trigger
    - Role: Entry point capturing user-submitted contact info (Name, Email, Company, Job Title, Message).
    - Configuration: Form fields defined with required flags; webhook URL auto-generated.
    - Outputs: Lead data JSON.
    - Edge Cases: Form submission errors, missing required fields, webhook failures.

  - **Log Lead to Sheet**
    - Type: Google Sheets (Append)
    - Role: Append raw lead data into specified Google Sheet “Leads” with columns for Name, Email, Company, Message, Job Title, Date.
    - Configuration: Uses Google Sheets credentials; append operation.
    - Inputs: Output from Input Form.
    - Outputs: Confirmation of append.
    - Edge Cases: Sheet ID misconfiguration, API quota limits, network errors.

  - **Get all columns**
    - Type: Google Sheets (Read)
    - Role: Reads all rows from “Leads” sheet to enable processing of the latest entry.
    - Configuration: Reads entire sheet by name and document ID.
    - Input: From Log Lead to Sheet.
    - Output: Full sheet data.
    - Edge Cases: Large data volume causing timeout.

  - **Return latest column**
    - Type: Code
    - Role: Extracts the last row (latest lead) from sheet data for further enrichment.
    - Configuration: JavaScript code selecting last item in the input array.
    - Input: Output of Get all columns.
    - Output: Single latest row.
    - Edge Cases: Empty sheet, data parsing errors.

---

### 2.2 AI Initial Response Agent

- **Overview:** Evaluates incoming lead message to determine if it can be answered automatically, requires escalation, or no response.

- **Nodes Involved:**
  - Structured Output Parser
  - OpenAI Chat Model
  - AI Answer Agent
  - Send Answer or Escalate

- **Node Details:**

  - **Structured Output Parser**
    - Type: LangChain Output Parser (Structured)
    - Role: Parses AI output into structured JSON (Sending To, Subject, Message).
    - Configuration: JSON schema example provided.
    - Connections: Feeds into AI Answer Agent as parser.

  - **OpenAI Chat Model**
    - Type: LangChain OpenAI Chat LM (gpt-4o-mini)
    - Role: Processes lead text for initial AI response analysis.
    - Configuration: Model set to gpt-4o-mini; no special options.
    - Output: AI response text.

  - **AI Answer Agent**
    - Type: LangChain Agent with output parser
    - Role: Decides if the lead inquiry can be answered, escalated, or ignored based on a system prompt with decision rules.
    - Configuration: System prompt defines criteria for when to answer, escalate, or not respond; output format JSON.
    - Inputs: Lead message details.
    - Outputs: JSON with email response details or escalation.
    - Edge Cases: Misclassification, model timeouts, API key issues.

  - **Send Answer or Escalate**
    - Type: Gmail Send
    - Role: Sends the AI-generated email or escalation to appropriate recipient.
    - Configuration: Email fields dynamically set from AI output.
    - Inputs: Output from AI Answer Agent.
    - Outputs: Email delivery confirmation.
    - Edge Cases: Gmail auth errors, invalid email addresses, send failures.

---

### 2.3 AI Lead Enrichment

- **Overview:** Enriches lead data by gathering company metadata and extracting pain points from the message using AI with web search capabilities.

- **Nodes Involved:**
  - OpenAI Search Model
  - AI Lead Enrichment
  - Define ICP and Lead Criteria

- **Node Details:**

  - **OpenAI Search Model**
    - Type: LangChain OpenAI Chat LM (gpt-4o-mini)
    - Role: Provides AI with search-enhanced capabilities for enrichment.
    - Configuration: Model gpt-4o-mini; no extra options.

  - **AI Lead Enrichment**
    - Type: LangChain Agent
    - Role: Generates enriched JSON with “Number of Employees”, “Industry”, “Geography”, “Annual Revenue”, “Technology”, and “Pain Points” extracted from message plus research.
    - Configuration: System prompt instructs to produce strict JSON output with no extraneous text.
    - Input: Lead company and message.
    - Output: JSON with enriched fields.
    - Edge Cases: Incomplete data, parsing errors, search API failures.

  - **Define ICP and Lead Criteria**
    - Type: Set
    - Role: Stores static text defining Ideal Customer Profile criteria and lead qualification categories (“demo-ready”, “nurture”, “drop”).
    - Configuration: Multi-line strings defining criteria with examples.
    - Output: Used downstream in classification.

---

### 2.4 AI Lead Classification

- **Overview:** Classifies leads into one of three categories (demo-ready, nurture, drop) based on enriched data and ICP criteria using GPT-4o.

- **Nodes Involved:**
  - AI Lead Classifier
  - OpenAI Chat
  - If demo-ready not empty
  - If nurture not empty
  - If drop not empty

- **Node Details:**

  - **AI Lead Classifier**
    - Type: LangChain Text Classifier
    - Role: Assigns exactly one classification based on company info, message, and ICP.
    - Configuration: Multi-category system prompt referencing ICP; categories are demo-ready, nurture, drop with descriptions from Set node.
    - Input: Combined data from Input Form and AI Lead Enrichment.
    - Output: Classification category text.
    - Edge Cases: Model ambiguity, no classification output.

  - **OpenAI Chat**
    - Type: LangChain OpenAI Chat LM (gpt-4o)
    - Role: (Appears unused or placeholder for enhanced classification or follow-up; linked to AI Lead Classifier input)
    - Configuration: Model gpt-4o.

  - **If demo-ready not empty**
    - Type: If
    - Role: Checks if lead classified as demo-ready; routes accordingly.
    - Configuration: Condition checks if classification output string exists.

  - **If nurture not empty**
    - Type: If
    - Role: Checks if lead classified as nurture; routes accordingly.

  - **If drop not empty**
    - Type: If
    - Role: Checks if lead classified as drop; routes accordingly.

---

### 2.5 Lead Follow-Up Sequences

- **Overview:** For each classification, executes tailored follow-up actions including data processing, email sending, calendar events, and updating Google Sheets.

- **Nodes Involved:**

  - Demo-Ready Path:
    - Demo-ready Data Processing (Code)
    - Update Sheet - Demo Ready
    - Schedule Demo - High Intent
    - Sticky Note, Wait nodes (email timing not applicable here)

  - Nurture Path:
    - Nurture Data Processing (Code)
    - Update Sheet - Nurture
    - Wait 1 Day (1), Send Nurture Email - Resources
    - Wait 1 Day (2), Send Nurture Email - Event
    - Wait 1 Day (3), Schedule Demo - After Nurture

  - Drop Path:
    - Drop Data Processing (Code)
    - Update Sheet - Drop
    - Send Drop Message

- **Node Details:**

  - **Demo-ready Data Processing**
    - Type: Code
    - Role: Parses AI enrichment output JSON and tags lead as “demo-ready”.
    - Edge Cases: JSON parse failures.

  - **Update Sheet - Demo Ready**
    - Type: Google Sheets (Update)
    - Role: Updates lead row with enrichment data and classification.
    - Configuration: Matches on row_number from latest sheet row.
    - Edge Cases: Update conflicts, row_number mismatch.

  - **Schedule Demo - High Intent**
    - Type: Google Calendar Event
    - Role: Creates a demo calendar event for the lead with Google Meet link.
    - Configuration: Schedules next business day at 12 PM - 1 PM, with fallback for weekends.
    - Edge Cases: Calendar permission issues, invalid email.

  - **Nurture Data Processing**
    - Type: Code
    - Role: Parses AI output for nurture leads and tags accordingly.

  - **Update Sheet - Nurture**
    - Type: Google Sheets (Update)
    - Role: Updates nurture leads’ enrichment and classification data.

  - **Wait 1 Day (1), (2), (3)**
    - Type: Wait
    - Role: Delays between nurture emails and scheduling (1 day each).
    - Edge Cases: Workflow pause persistence issues.

  - **Send Nurture Email - Resources**
    - Type: Gmail Send
    - Role: Sends initial nurture email with resources and benefits.
    - Configuration: Personalized using lead data.
    - Edge Cases: Email send failures.

  - **Send Nurture Email - Event**
    - Type: Gmail Send
    - Role: Sends webinar invitation or event nurture email.
    - Edge Cases: Same as above.

  - **Schedule Demo - After Nurture**
    - Type: Google Calendar Event
    - Role: Schedules demo after nurture sequence with personalized description.
    - Edge Cases: Same as Schedule Demo - High Intent.

  - **Drop Data Processing**
    - Type: Code
    - Role: Parses AI output for drop leads and tags accordingly.

  - **Update Sheet - Drop**
    - Type: Google Sheets (Update)
    - Role: Updates drop leads’ data in the sheet.

  - **Send Drop Message**
    - Type: Gmail Send
    - Role: Sends polite rejection email to drop leads.
    - Configuration: Email content explains non-qualification with encouragement to re-contact.
    - Edge Cases: Email send failures.

---

### 2.6 Sticky Notes: Configuration & Guidance

Sticky notes are used extensively for:

- Explaining lead classification logic and improvement ideas.
- Providing detailed instructions on setting wait times, email templates, and calendar configurations for nurture and demo-ready flows.
- Documenting prerequisites and setup steps for form and Google Sheets integration.
- Outlining AI Answer Agent behavior and escalation policies.
- Suggestions for future workflow enhancements like CRM integration, multi-channel follow-ups, and lead scoring.

---

## 3. Summary Table

| Node Name                  | Node Type                           | Functional Role                            | Input Node(s)             | Output Node(s)                      | Sticky Note                                                                                  |
|----------------------------|-----------------------------------|-------------------------------------------|---------------------------|------------------------------------|----------------------------------------------------------------------------------------------|
| Input Form                 | Form Trigger                      | Lead data entry via web form               | None                      | AI Answer Agent, Log Lead to Sheet | 📥 Input Form & Logging: Setup instructions, Google Sheet columns, webhook URL               |
| Log Lead to Sheet          | Google Sheets (Append)            | Append raw lead data to Google Sheet       | Input Form                | Get all columns                    |                                                                                              |
| Get all columns            | Google Sheets (Read)              | Read entire “Leads” sheet                   | Log Lead to Sheet          | Return latest column               |                                                                                              |
| Return latest column       | Code                             | Extract latest lead row                      | Get all columns            | AI Lead Enrichment                |                                                                                              |
| Structured Output Parser   | LangChain Output Parser           | Parse AI response JSON                       | OpenAI Chat Model          | AI Answer Agent                   |                                                                                              |
| OpenAI Chat Model          | LangChain OpenAI Chat LM          | Base AI model for initial answer analysis   |                            | AI Answer Agent                   |                                                                                              |
| AI Answer Agent            | LangChain Agent                   | Auto-answer, escalate, or ignore inquiry    | Input Form, Structured Output Parser | Send Answer or Escalate         | 💬 AI Answer Agent: Configuration, escalation email, response instructions                   |
| Send Answer or Escalate    | Gmail Send                       | Send AI-generated or escalation email       | AI Answer Agent            | None                             |                                                                                              |
| OpenAI Search Model        | LangChain OpenAI Chat LM          | AI with search for enrichment                |                            | AI Lead Enrichment                |                                                                                              |
| AI Lead Enrichment         | LangChain Agent                   | Enrich lead data with company info          | Return latest column       | Define ICP and Lead Criteria      | 📊 Lead Enrichment: Setup, OpenAI key, future data sources                                   |
| Define ICP and Lead Criteria| Set                              | Store ICP and lead qualification criteria   | AI Lead Enrichment         | AI Lead Classifier                | 🤖 Lead Classification: Instructions to edit ICP and criteria                               |
| AI Lead Classifier         | LangChain Text Classifier          | Classify lead as demo-ready, nurture, or drop| Define ICP and Lead Criteria | If demo-ready not empty, If nurture not empty, If drop not empty |                                                                                              |
| OpenAI Chat                | LangChain OpenAI Chat LM          | Additional AI model (possibly for classification refinement) | AI Lead Classifier          | AI Lead Classifier                |                                                                                              |
| If demo-ready not empty    | If                               | Check if lead is demo-ready                  | AI Lead Classifier          | Demo-ready Data Processing        |                                                                                              |
| Demo-ready Data Processing | Code                             | Parse enrichment JSON and tag demo-ready    | If demo-ready not empty    | Update Sheet - Demo Ready, Schedule Demo - High Intent |                                                                                              |
| Update Sheet - Demo Ready  | Google Sheets (Update)            | Update lead data with enrichment, classification | Demo-ready Data Processing | None                             |                                                                                              |
| Schedule Demo - High Intent| Google Calendar Event             | Schedule demo event for demo-ready leads     | Demo-ready Data Processing | None                             | ✅ Demo-Ready Follow-Up: Calendar config, meeting duration, auto-scheduling                  |
| If nurture not empty       | If                               | Check if lead is nurture                      | AI Lead Classifier          | Nurture Data Processing           |                                                                                              |
| Nurture Data Processing    | Code                             | Parse enrichment JSON and tag nurture        | If nurture not empty       | Update Sheet - Nurture, Wait 1 Day (1) |                                                                                              |
| Update Sheet - Nurture     | Google Sheets (Update)            | Update lead data with enrichment, classification | Nurture Data Processing    | None                             |                                                                                              |
| Wait 1 Day (1)             | Wait                             | Delay before sending nurture email 1         | Nurture Data Processing    | Send Nurture Email - Resources    | 🎯 Nurture Follow-Up Sequence: Wait times, email content, calendar integration              |
| Send Nurture Email - Resources | Gmail Send                   | Send first nurture email with resources      | Wait 1 Day (1)             | Wait 1 Day (2)                   |                                                                                              |
| Wait 1 Day (2)             | Wait                             | Delay before sending nurture email 2         | Send Nurture Email - Resources | Send Nurture Email - Event       |                                                                                              |
| Send Nurture Email - Event | Gmail Send                       | Send webinar invitation or educational email | Wait 1 Day (2)             | Wait 1 Day (3)                   |                                                                                              |
| Wait 1 Day (3)             | Wait                             | Delay before scheduling nurture demo          | Send Nurture Email - Event | Schedule Demo - After Nurture    |                                                                                              |
| Schedule Demo - After Nurture | Google Calendar Event          | Schedule demo after nurture sequence          | Wait 1 Day (3)             | None                             |                                                                                              |
| If drop not empty          | If                               | Check if lead is drop                          | AI Lead Classifier          | Drop Data Processing, Send Drop Message |                                                                                              |
| Drop Data Processing       | Code                             | Parse enrichment JSON and tag drop             | If drop not empty          | Update Sheet - Drop              |                                                                                              |
| Update Sheet - Drop        | Google Sheets (Update)            | Update lead data with drop classification      | Drop Data Processing       | None                             |                                                                                              |
| Send Drop Message          | Gmail Send                       | Send polite rejection email                     | Drop Data Processing       | None                             | ❌ Drop Follow-Up: Drop message config, keep door open, website link                        |
| Sticky Notes (various)     | Sticky Note                      | Configuration guidance and workflow notes      | N/A                       | N/A                             | See section 2.6                                                                              |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add Input Form Node**
   - Type: Form Trigger
   - Configure form fields: Name, Email, Company, Job Title (all required), Message (textarea, required).
   - Save and note the webhook URL for external form integration.

3. **Add Log Lead to Sheet Node**
   - Type: Google Sheets (Append)
   - Connect Input Form output to this node.
   - Configure with your Google Sheets credentials.
   - Set Sheet ID and sheet name “Leads”.
   - Map columns: Date (submission time), Name, Mail (Email), Company, Message, Job Title.
   
4. **Add Get all columns Node**
   - Type: Google Sheets (Read)
   - Connect Log Lead to Sheet output.
   - Configure to read all rows from “Leads” sheet.

5. **Add Return latest column Node**
   - Type: Code
   - Connect Get all columns output.
   - Use JS code to return last row: `const lastRow = items[items.length - 1]; return [lastRow];`

6. **Add OpenAI Chat Model Node (for AI initial response)**
   - Type: LangChain OpenAI Chat
   - Set model to “gpt-4o-mini”.
   - Connect Input Form output to its input if needed.

7. **Add Structured Output Parser Node**
   - Type: LangChain Output Parser Structured
   - Configure JSON schema with fields: Sending To, Subject, Message.

8. **Add AI Answer Agent Node**
   - Type: LangChain Agent
   - Connect OpenAI Chat Model output as language model and Structured Output Parser as output parser.
   - Configure system message prompt to classify inquiries into answerable, escalation, or no response with detailed rules.
   - Use dynamic expressions to pass lead message, name, job title, company, email.

9. **Add Send Answer or Escalate Node**
   - Type: Gmail Send
   - Configure Gmail OAuth2 credentials.
   - Connect AI Answer Agent output.
   - Use dynamic expressions to set “Send To”, “Subject”, “Message”.

10. **Add OpenAI Search Model Node**
    - Type: LangChain OpenAI Chat
    - Set model to “gpt-4o-mini”.
    - Connect Return latest column output.

11. **Add AI Lead Enrichment Node**
    - Type: LangChain Agent
    - Connect OpenAI Search Model output as language model.
    - Prompt to enrich company lead info and extract pain points, output strictly in JSON per schema.
    - Connect Return latest column as input data.

12. **Add Define ICP and Lead Criteria Node**
    - Type: Set
    - Connect AI Lead Enrichment output.
    - Fill string fields with your Ideal Customer Profile criteria and lead classification rules for demo-ready, nurture, drop.

13. **Add AI Lead Classifier Node**
    - Type: LangChain Text Classifier
    - Connect Define ICP and Lead Criteria output.
    - Input text dynamically composed from lead company, message, and enriched data.
    - Define three categories with descriptions from Set node.
    - Use system prompt instructing single classification selection.

14. **Add three If Nodes:**
    - If demo-ready not empty: check if classification equals “demo-ready”.
    - If nurture not empty: check if classification equals “nurture”.
    - If drop not empty: check if classification equals “drop”.

15. **For Demo-Ready Path:**
    - Add Demo-ready Data Processing Node (Code) to parse JSON output and tag “demo-ready”.
    - Add Update Sheet - Demo Ready Node (Google Sheets Update) to write enrichment and classification data back to the sheet.
    - Add Schedule Demo - High Intent Node (Google Calendar) to create demo event next business day 12-1 PM with Google Meet.
    - Connect nodes sequentially from If demo-ready to Data Processing to Update Sheet to Calendar.

16. **For Nurture Path:**
    - Add Nurture Data Processing Node (Code) similar to demo-ready.
    - Add Update Sheet - Nurture Node (Google Sheets Update).
    - Add three Wait Nodes of 1 day each.
    - Add Send Nurture Email - Resources Node (Gmail) after first wait.
    - Add Send Nurture Email - Event Node (Gmail) after second wait.
    - Add Schedule Demo - After Nurture Node (Google Calendar) after third wait.
    - Connect nodes in order: If nurture → Data Processing → Update Sheet → Wait 1 → Email Resources → Wait 2 → Email Event → Wait 3 → Schedule Demo.

17. **For Drop Path:**
    - Add Drop Data Processing Node (Code).
    - Add Update Sheet - Drop Node (Google Sheets Update).
    - Add Send Drop Message Node (Gmail) with polite rejection email.
    - Connect sequentially: If drop → Data Processing → Update Sheet → Send Drop Message.

18. **Add Sticky Notes**
    - Add sticky notes for configuration instructions for each major block as per original workflow.

19. **Credentials Setup**
    - Set up Google OAuth2 credentials for Sheets, Gmail, Calendar nodes.
    - Configure OpenAI API credentials for LangChain nodes.
    - Validate all nodes have appropriate access.

20. **Test Workflow**
    - Submit sample leads via the form webhook.
    - Observe logs, email sending, calendar events, and sheet updates.
    - Adjust ICP criteria and email templates as needed.

---

## 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                     | Context or Link                                                                                      |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Workflow automates B2B lead qualification with AI and Google Workspace tools, uses GPT-4o for enrichment and classification.                                                                                                     | Workflow Overview                                                                                   |
| Sticky notes provide detailed instructions on configuring wait times, email templates, calendar event settings, and AI prompt tuning.                                                                                          | Workflow Sticky Notes                                                                               |
| Future improvements suggest CRM integration, lead scoring, multi-channel follow-ups, and enriched data sources like LinkedIn or Clearbit.                                                                                       | Sticky Notes and Node Comments                                                                     |
| AI Answer Agent escalation email must be replaced with your actual sales team email for proper escalation.                                                                                                                      | AI Answer Agent Sticky Note                                                                         |
| Google Sheet “Leads” must have specified columns and your Google Sheet ID must be configured in all relevant nodes.                                                                                                            | Input Form & Logging Sticky Note                                                                    |
| Gmail nodes send plain text emails personalized with lead data; customize messages and subjects to fit your branding and tone.                                                                                                  | Send Email Nodes                                                                                     |
| Calendar events are scheduled intelligently to avoid weekends, defaulting to next business day at noon with Google Meet conferencing enabled.                                                                                   | Schedule Demo Sticky Notes                                                                           |
| The workflow respects n8n version compatibility with nodes using the latest stable versions as of the knowledge cutoff.                                                                                                         | Node version notes                                                                                   |
| Documentation and improvements inspired by blog posts and community examples on AI-powered lead management workflows.                                                                                                          | External inspiration (not linked directly here)                                                    |

---

**Disclaimer:** The provided content originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.

---
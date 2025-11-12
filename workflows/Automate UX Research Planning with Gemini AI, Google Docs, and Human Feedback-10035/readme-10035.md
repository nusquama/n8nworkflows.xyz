Automate UX Research Planning with Gemini AI, Google Docs, and Human Feedback

https://n8nworkflows.xyz/workflows/automate-ux-research-planning-with-gemini-ai--google-docs--and-human-feedback-10035


# Automate UX Research Planning with Gemini AI, Google Docs, and Human Feedback

---

### 1. Workflow Overview

This workflow automates the creation, refinement, and delivery of a UX research plan by integrating AI-driven question generation and editing with human feedback loops. It targets UX researchers and designers who want to efficiently generate tailored research questions, refine them collaboratively, recommend suitable UX research methods, and produce a polished research plan document.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Collects organizational and research context from a user-submitted form.
- **1.2 AI-Driven Research Question Generation:** Uses Google Gemini AI and LangChain agents to generate an initial set of research questions based on the input.
- **1.3 User Review and Iterative Refinement:** Sends generated questions to the user via email for approval or edits, processes feedback, and iteratively improves questions with AI assistance.
- **1.4 Research Method Recommendations:** AI suggests appropriate UX research methods for each approved question.
- **1.5 Research Report Preparation:** Compiles the final research questions and methods into a well-structured HTML report suitable for Google Docs.
- **1.6 Final Editing and Delivery:** Incorporates user feedback on the report, refines it, creates a Google Docs document, and sends it for final approval.

This systematic loop ensures high-quality, context-relevant research plans that combine AI creativity with human expertise.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
Collects essential background information about the organization, product, research goal, and preferred email address via a form submission.

- **Nodes Involved:**  
  - On form submission (Form Trigger)  
  - Sticky Note (Information about organization)

- **Node Details:**

  - **On form submission**  
    - Type: Form Trigger  
    - Configuration: Custom form titled "Provide Context for Writing a Research Plan" with four required fields: organization type, product/service description, research learning goals, and preferred email address.  
    - Inputs: External form submission webhook  
    - Outputs: JSON object containing user responses  
    - Potential Failures: Missing required fields, webhook connectivity issues.

  - **Sticky Note (Information about organization)**  
    - Type: Sticky Note (documentation aid)  
    - Content: Visual reminder of the block's purpose, no functional role.

#### 2.2 AI-Driven Research Question Generation

- **Overview:**  
Generates 10 initial UX research questions based on the collected input using Google Gemini AI and a LangChain agent, then parses these questions into a structured JSON object.

- **Nodes Involved:**  
  - AI Agent: Question Maker  
  - Gemini (Google Gemini AI node)  
  - Separate Questions (Code node)  
  - Sticky Note (Using AI to Write a Research Plan)

- **Node Details:**

  - **AI Agent: Question Maker**  
    - Type: LangChain Agent  
    - Configuration: Uses a prompt that takes the form responses to generate 10 research questions in strict JSON format.  
    - Key Expressions: References form inputs with expressions like `{{ $json['What type of organization do you work for?'] }}`.  
    - Input: Form submission data  
    - Output: Text containing JSON with 10 questions  
    - Failure Modes: Parsing errors, AI service errors, malformed JSON output.

  - **Gemini**  
    - Type: Google Gemini AI node (PaLM API)  
    - Configuration: Connected to Google Palm API credentials; serves as underlying AI model provider for the agent.  
    - Input: Invoked by AI Agent node  
    - Output: AI-generated content.

  - **Separate Questions**  
    - Type: Code Node (JavaScript)  
    - Configuration: Parses the JSON string output from the AI Agent, removing markdown wrappers, and outputs a clean JSON object with questions.  
    - Input: Raw JSON string from AI Agent  
    - Output: Parsed questions as JSON object  
    - Failure Modes: JSON parse exceptions if AI output is malformed.

  - **Sticky Note (Using AI to Write a Research Plan)**  
    - Documentation, no functional role.

#### 2.3 User Review and Iterative Refinement

- **Overview:**  
Sends generated questions to the user via email for review with options to approve, reject, or request edits. Processes user feedback, combines it with questions, and iteratively refines questions using AI agents. This loop may repeat to finalize questions.

- **Nodes Involved:**  
  - Question Approval (Gmail node)  
  - Combine Questions (Code node)  
  - AI Agent: Question Editor  
  - Edit Approval (Gmail node)  
  - AI Agent: Second Editor  
  - Sticky Notes (Questions Checked, Edit Questions, Questions Regenerated)  

- **Node Details:**

  - **Question Approval**  
    - Type: Gmail Node (Send and Wait)  
    - Configuration: Sends an email with research questions and a custom form allowing the user to approve, disapprove, or mark questions for editing, with text input for edit suggestions.  
    - Inputs: Questions JSON  
    - Outputs: User responses via email  
    - Failure Modes: Email delivery failure, user non-response, parsing form responses.

  - **Combine Questions**  
    - Type: Code Node  
    - Configuration: Merges user feedback with original questions, forming a combined JSON object including question text, status (approved/disapproved/should edit), and edit suggestions.  
    - Input: User form response  
    - Output: Structured combined data for further processing.

  - **AI Agent: Question Editor**  
    - Type: LangChain Agent  
    - Configuration: Refines or rewrites questions marked for editing based on user feedback, keeps approved questions unchanged, and avoids rejected question formats.  
    - Input: Combined question and feedback data, original form input for context  
    - Output: Refined list of questions in a numbered list without markdown  
    - Failure Modes: AI service errors, response consistency.

  - **Edit Approval**  
    - Type: Gmail Node (Send and Wait)  
    - Configuration: Sends refined questions for a second round of user feedback, asking which questions are still not approved and reasons for rejection.  
    - Input: Refined questions  
    - Output: User rejection feedback.

  - **AI Agent: Second Editor**  
    - Type: LangChain Agent  
    - Configuration: Rewrites rejected questions using user-provided rejection reasons, aiming for improved quality.  
    - Input: User rejection reasons and list of rejected questions  
    - Output: Final refined questions list.

  - **Sticky Notes**  
    - Provide contextual annotations such as "Questions are checked by the user", "Edit the questions based on user feedback", and "Questions are regenerated".

#### 2.4 Research Method Recommendations

- **Overview:**  
For each finalized research question, AI recommends the most suitable UX research methods and provides rationale, producing a clear, concise list.

- **Nodes Involved:**  
  - AI Agent: Request Methods  
  - Google Gemini Chat (Google Gemini AI node)  
  - Sticky Note (Design methods are proposed for each question)

- **Node Details:**

  - **AI Agent: Request Methods**  
    - Type: LangChain Agent  
    - Configuration: Given the research context and finalized questions, recommends UX research methods (e.g., interviews, usability testing) with brief justification for each.  
    - Input: Final question list, original form input for context  
    - Output: Plain text numbered list with question, method(s), and rationale  
    - Failure Modes: AI availability, output formatting errors.

  - **Google Gemini Chat**  
    - Type: Google Gemini AI node  
    - Configuration: Provides AI inference backend for the agent.

#### 2.5 Research Report Preparation

- **Overview:**  
Transforms the UX research questions and recommended methods into a clean, well-structured HTML report ready for Google Docs import.

- **Nodes Involved:**  
  - AI Agent (Report Formatter)  
  - Gemini ChatAssis (Google Gemini AI node)  
  - Create a Report (Google Docs node)  
  - HTTP Request: Report (HTTP Request node)  
  - Sticky Note (Prepare the Report)

- **Node Details:**

  - **AI Agent**  
    - Type: LangChain Agent  
    - Configuration: Converts raw content (questions and methods) into a visually appealing HTML report with semantic HTML tags (<h1>, <h2>, <p>, etc.) and minimal inline styling for professional presentation.  
    - Input: Text output from AI Agent: Request Methods  
    - Output: HTML-formatted report content.

  - **Gemini ChatAssis**  
    - Type: Google Gemini AI node  
    - Provides AI backend for the formatting agent.

  - **Create a Report**  
    - Type: Google Docs Node  
    - Configuration: Creates a new Google Doc titled with a timestamped "Research Plan", stored in the default folder.  
    - Input: Triggered after methods recommendation  
    - Output: Document metadata including document ID.

  - **HTTP Request: Report**  
    - Type: HTTP Request Node  
    - Configuration: Posts the HTML report content to a Google Apps Script web app endpoint that appends the content to the created Google Doc by document ID.  
    - Input: HTML output and Google Doc ID  
    - Failure Modes: Network errors, Google Apps Script endpoint errors.

  - **Sticky Note**  
    - Describes the block's function: preparing the report.

#### 2.6 Final Editing and Delivery

- **Overview:**  
Sends the generated report to the user for feedback, applies requested changes via AI, updates the Google Doc, and sends a final approval request.

- **Nodes Involved:**  
  - Report Approval (Gmail node)  
  - AI Agent: Report Editor  
  - Google Gemini Chat (Google Gemini AI node)  
  - Create the Final Report (Google Docs node)  
  - HTTP Request: Edited Report (HTTP Request node)  
  - Sticky Notes (Get and Send the Report, Create the Final Report)

- **Node Details:**

  - **Report Approval**  
    - Type: Gmail Node (Send and Wait)  
    - Configuration: Sends an email to the user including a link to the Google Doc report and a form field asking for feedback on changes needed.  
    - Input: Google Doc ID from report creation  
    - Output: User feedback on report changes.

  - **AI Agent: Report Editor**  
    - Type: LangChain Agent  
    - Configuration: Revises the report HTML based on user feedback, maintaining structure and clarity.  
    - Input: Original report HTML and user feedback  
    - Output: Updated report HTML.

  - **Google Gemini Chat**  
    - Type: Google Gemini AI node  
    - Provides AI backend for the report editor.

  - **Create the Final Report**  
    - Type: Google Docs Node  
    - Configuration: Creates a new Google Doc with a timestamped "Final Research Plan" title.  
    - Input: Triggered after report editing  
    - Output: Final document metadata.

  - **HTTP Request: Edited Report**  
    - Type: HTTP Request Node  
    - Configuration: Posts the edited report HTML to the Google Apps Script endpoint to append it to the final Google Doc.  
    - Input: Edited HTML and final document ID.

  - **Sticky Notes**  
    - Document the stage: "Get and Send the Report" and "Create the Final Report".

---

### 3. Summary Table

| Node Name               | Node Type                              | Functional Role                                              | Input Node(s)              | Output Node(s)                  | Sticky Note                                                                                   |
|-------------------------|--------------------------------------|--------------------------------------------------------------|----------------------------|-------------------------------|----------------------------------------------------------------------------------------------|
| On form submission      | Form Trigger                         | Collect user input about organization and research goals     | External webhook           | AI Agent: Question Maker       |                                                                                              |
| Gemini                  | Google Gemini AI                     | AI backend for question generation                           | AI Agent: Question Maker   | AI Agent: Question Maker        |                                                                                              |
| AI Agent: Question Maker | LangChain Agent                     | Generates 10 initial UX research questions                   | On form submission         | Separate Questions              |                                                                                              |
| Separate Questions      | Code Node                            | Parses AI output JSON string into structured JSON            | AI Agent: Question Maker   | Question Approval              |                                                                                              |
| Question Approval       | Gmail (Send and Wait)                | Sends questions to user for approval or edits                | Separate Questions         | Combine Questions              |                                                                                              |
| Combine Questions       | Code Node                           | Combines user feedback with questions                         | Question Approval          | AI Agent: Question Editor       |                                                                                              |
| AI Agent: Question Editor| LangChain Agent                    | Refines questions based on user feedback                      | Combine Questions          | Edit Approval                 |                                                                                              |
| Edit Approval           | Gmail (Send and Wait)                | Sends refined questions for second user review               | AI Agent: Question Editor  | AI Agent: Second Editor         |                                                                                              |
| AI Agent: Second Editor | LangChain Agent                    | Rewrites rejected questions using user reasons               | Edit Approval             | AI Agent: Request Methods       |                                                                                              |
| Gemini Google           | Google Gemini AI                    | AI backend for question editing agent                         | AI Agent: Question Editor  | AI Agent: Question Editor       |                                                                                              |
| Google Gemini Chat      | Google Gemini AI                    | AI backend for method recommendation agent                   | AI Agent: Second Editor    | AI Agent: Request Methods       |                                                                                              |
| AI Agent: Request Methods| LangChain Agent                    | Recommends UX research methods for questions                  | AI Agent: Second Editor    | AI Agent                      |                                                                                              |
| AI Agent                | LangChain Agent                    | Formats research questions and methods into HTML report      | AI Agent: Request Methods  | HTTP Request: Report            |                                                                                              |
| Gemini ChatAssis        | Google Gemini AI                    | AI backend for report formatting                              | AI Agent                  | AI Agent                      |                                                                                              |
| Create a Report         | Google Docs Node                   | Creates initial Google Doc for research plan                  | AI Agent: Request Methods  | HTTP Request: Report            |                                                                                              |
| HTTP Request: Report    | HTTP Request Node                  | Appends HTML report content to Google Doc                     | Create a Report           | Report Approval                |                                                                                              |
| Report Approval         | Gmail (Send and Wait)                | Sends report link and collects user feedback                  | HTTP Request: Report       | AI Agent: Report Editor         |                                                                                              |
| AI Agent: Report Editor | LangChain Agent                    | Revises report based on feedback                              | Report Approval           | Create the Final Report         |                                                                                              |
| Google GeminiChat       | Google Gemini AI                    | AI backend for report editor agent                            | AI Agent: Report Editor     | AI Agent: Report Editor         |                                                                                              |
| Create the Final Report | Google Docs Node                   | Creates the final Google Doc with revised report              | AI Agent: Report Editor    | HTTP Request: Edited Report     |                                                                                              |
| HTTP Request: Edited Report | HTTP Request Node              | Appends edited report HTML to final Google Doc                | Create the Final Report    |                              |                                                                                              |
| Sticky Note             | Sticky Note                        | Various descriptive notes throughout the workflow             |                            |                               | "Get Information about Organisation"                                                        |
| Sticky Note1            | Sticky Note                        |                                                              |                            |                               | "Using AI to Write a Research Plan"                                                        |
| Sticky Note2            | Sticky Note                        |                                                              |                            |                               | "Questions are checked by the user"                                                       |
| Sticky Note3            | Sticky Note                        |                                                              |                            |                               | "Edit the questions based on the user's feedback and be rechecked by the user"            |
| Sticky Note4            | Sticky Note                        |                                                              |                            |                               | "Questions are regenerated"                                                               |
| Sticky Note5            | Sticky Note                        |                                                              |                            |                               | "Design methods are proposed for each question"                                          |
| Sticky Note6            | Sticky Note                        |                                                              |                            |                               | "Prepare the Report"                                                                      |
| Sticky Note7            | Sticky Note                        |                                                              |                            |                               | "Get and Send the Report"                                                                 |
| Sticky Note8            | Sticky Note                        |                                                              |                            |                               | "Edit the Report"                                                                         |
| Sticky Note9            | Sticky Note                        |                                                              |                            |                               | "Create the Final Report"                                                                 |
| Sticky Note10           | Sticky Note                        |                                                              |                            |                               | "Planning Research Workflow" (detailed summary of the whole workflow)                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node ("On form submission")**  
   - Type: Form Trigger  
   - Title: "Provide Context for Writing a Research Plan"  
   - Fields (all required):  
     - Textarea: "What type of organization do you work for?"  
     - Textarea: "What kind of product, service, or experience does your company provide?"  
     - Textarea: "What do you want to learn about your users or the problem you are addressing?"  
     - Email: "Which email address would you prefer to receive the email at?"  
   - Set webhook to accept submissions.

2. **Add a LangChain AI Agent Node ("AI Agent: Question Maker")**  
   - Connect input from "On form submission"  
   - Use Google Gemini AI as backend (set credentials for Google Palm API)  
   - Prompt: Generate exactly 10 UX research questions in JSON format using the form inputs as variables  
   - Ensure output is text containing JSON.

3. **Add a Code Node ("Separate Questions")**  
   - Input: Output from "AI Agent: Question Maker"  
   - Code: Parse the JSON string, remove markdown code fences, and output clean JSON of questions.

4. **Add a Gmail Node ("Question Approval")**  
   - Operation: Send and Wait for response  
   - Send to: email collected in form submission  
   - Email subject: Request for Approval of Research Questions  
   - Message body: Include polite request and list the 10 questions with radio buttons for "Approved", "Disapproved", "Should be edited" and textareas for edit suggestions per question.

5. **Add a Code Node ("Combine Questions")**  
   - Input: User response data from "Question Approval"  
   - Code: Combine questions, statuses, and edit suggestions into a structured JSON for AI processing.

6. **Add LangChain Agent Node ("AI Agent: Question Editor")**  
   - Input: Output from "Combine Questions" plus original form context  
   - Backend: Google Gemini AI  
   - Prompt: Refine or rewrite questions marked for editing, keep approved unchanged, avoid rejected styles, output numbered list of final questions.

7. **Add Gmail Node ("Edit Approval")**  
   - Send and Wait  
   - Email to user with refined questions from "AI Agent: Question Editor"  
   - Ask which questions are still disapproved and reasons why.

8. **Add LangChain Agent Node ("AI Agent: Second Editor")**  
   - Input: User feedback from "Edit Approval"  
   - Backend: Google Gemini AI  
   - Prompt: Rewrite rejected questions using user-provided reasons, output numbered list.

9. **Add LangChain Agent Node ("AI Agent: Request Methods")**  
   - Input: Final question list from "AI Agent: Second Editor" plus original context  
   - Backend: Google Gemini AI  
   - Prompt: For each question, recommend UX research methods with concise rationale, output numbered plain text list.

10. **Add LangChain Agent Node ("AI Agent") for Report Formatting**  
    - Input: Output from "AI Agent: Request Methods"  
    - Backend: Google Gemini AI  
    - Prompt: Format content into polished, well-structured semantic HTML report, suitable for Google Docs.

11. **Add Google Docs Node ("Create a Report")**  
    - Action: Create document  
    - Title: Timestamped "Research Plan"  
    - Folder: Default or specify folder  
    - Input: Trigger after methods recommendation.

12. **Add HTTP Request Node ("HTTP Request: Report")**  
    - POST request to Google Apps Script endpoint  
    - Body parameters: docId (from created document), mode: append, HTML: report content from AI Agent  
    - Triggered after document creation.

13. **Add Gmail Node ("Report Approval")**  
    - Send and Wait  
    - Email to user with Google Doc link  
    - Ask for feedback on report changes.

14. **Add LangChain Agent Node ("AI Agent: Report Editor")**  
    - Input: Original report HTML and user feedback  
    - Backend: Google Gemini AI  
    - Prompt: Revise report based on feedback, maintaining clarity and structure.

15. **Add Google Docs Node ("Create the Final Report")**  
    - Create new Google Doc titled "Final Research Plan" with timestamp.

16. **Add HTTP Request Node ("HTTP Request: Edited Report")**  
    - POST updated report HTML to Google Apps Script endpoint, appending to final document.

17. **Connect all nodes in the order above according to input/output dependencies.**

18. **Set all required credentials:**  
    - Google Palm API for Gemini nodes  
    - Gmail OAuth2 for Gmail nodes  
    - Google Docs OAuth2 for Docs nodes.

19. **Add Sticky Notes at relevant workflow sections for documentation and clarity.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                | Context or Link                                                                                                   |
|---------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| This workflow automates UX research planning using AI and human feedback loops integrating Google Gemini, Gmail, and Google Docs.           | Sticky Note10 content summarizing the entire workflow logic                                                       |
| The Google Apps Script endpoint URL used for appending content to Google Docs: https://script.google.com/macros/s/AKfycbycCRkQRMw2hQPRKvBq-8fPSTlrWsHt3rNd5qjuCytZavI66m1ovif4HEoW9TtuUaar/exec | HTTP Request node configuration                                                                                   |
| The workflow uses multi-round email forms (Gmail nodes with "Send and Wait") to capture detailed user feedback for iterative AI refinement. | Gmail node configuration notes                                                                                    |
| AI prompt engineering carefully structures the input and output format to ensure JSON compliance and plain text lists for reliable automation. | Prompts in LangChain Agent Nodes                                                                                  |
| Timestamp-based naming conventions for Google Docs ensure unique, chronologically ordered research plans.                                   | Google Docs node title parameter formatting                                                                       |

---

**Disclaimer:**  
The provided text is extracted exclusively from an automated workflow developed using n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All manipulated data are legal and public.

---
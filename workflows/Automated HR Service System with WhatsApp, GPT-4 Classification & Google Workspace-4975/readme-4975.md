Automated HR Service System with WhatsApp, GPT-4 Classification & Google Workspace

https://n8nworkflows.xyz/workflows/automated-hr-service-system-with-whatsapp--gpt-4-classification---google-workspace-4975


# Automated HR Service System with WhatsApp, GPT-4 Classification & Google Workspace

### 1. Workflow Overview

This workflow is an **Automated HR Service System** designed for processing employee interactions over WhatsApp using AI-driven classification and Google Workspace integrations. The system targets HR-related use cases such as leave requests, attendance marking, HR FAQs, complaint escalation, and candidate shortlisting. It employs GPT-4 based language models for message classification, transcription, and response generation, integrating with Google Sheets for data storage, Google Calendar for scheduling, Gmail for email communication, and WhatsApp for user interaction.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Classification:** Receives WhatsApp messages, verifies message types, and classifies intent into one of five categories using a large language model (LLM).
- **1.2 Message Routing and Preprocessing:** Routes messages based on classification to different HR service agents and prepares data for AI processing.
- **1.3 Leave Management Agent:** Handles leave requests with approval logic and email notifications.
- **1.4 HR Chatbot Agent:** Answers general HR queries using AI and policy document embeddings.
- **1.5 Attendance Management Agent:** Processes location-based attendance marking with geofencing and timesheet updates.
- **1.6 Complaint/Request Email Agent:** Generates and sends emails for complaints or requests to department heads.
- **1.7 Candidate Shortlisting Agent:** Filters candidates against job descriptions, scores them, schedules interviews, and communicates results.
- **1.8 Final WhatsApp Response:** Consolidates agent outputs and sends appropriate WhatsApp replies to users.
- **1.9 Media Handling and Transcription:** Downloads and transcribes audio/image media messages for processing by AI agents.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Classification

**Overview:**  
Receives incoming WhatsApp messages, verifies message type (text, audio, image), and classifies user intent into one of five categories (leave request, HR FAQ, attendance, complaint escalation, or shortlisting request) using a large language model.

**Nodes Involved:**  
- WhatsApp Trigger  
- Delegatory LLM  
- Switch  
- Verify Message Type  
- Edit Fields / Edit Fields2 / Edit Fields3  
- download media / download media1  
- HTTP Request / HTTP Request1  

**Node Details:**

- **WhatsApp Trigger**  
  - Type: WhatsApp webhook trigger  
  - Role: Entry point receiving WhatsApp messages  
  - Config: Listens for message updates, uses WhatsApp OAuth credentials  
  - Inputs: External WhatsApp messages  
  - Outputs: Raw WhatsApp message JSON  
  - Failures: Connectivity, auth errors, webhook misconfiguration  

- **Delegatory LLM**  
  - Type: Langchain LLM Chain  
  - Role: Classifies message into category 1-5 strictly  
  - Config: Prompt instructs to analyze message type and body, outputting only a number (1-5)  
  - Inputs: WhatsApp message JSON  
  - Outputs: Classification number  
  - Edge Cases: Misclassification, unexpected input formats, empty messages  

- **Switch**  
  - Type: Switch (Router)  
  - Role: Routes flow based on classification number (1 to 5)  
  - Config: Routes to Leave Agent, HR Chatbot, Attendance, Complaint, or Shortlist Agent  
  - Inputs: Classification number  
  - Outputs: Branches for each category  
  - Edge Cases: Unexpected classification values, no output branch  

- **Verify Message Type**  
  - Type: Switch  
  - Role: Determines message content type (text, audio, image) to trigger appropriate processing  
  - Inputs: WhatsApp message JSON  
  - Outputs: Routes to transcription or direct text processing  
  - Edge Cases: Unsupported media types, missing fields  

- **Edit Fields / Edit Fields2 / Edit Fields3**  
  - Type: Set node  
  - Role: Normalizes or extracts specific JSON fields for downstream processing (e.g., message text)  
  - Inputs: WhatsApp JSON, OpenAI responses  
  - Outputs: Refined JSON with required fields  

- **download media / download media1**  
  - Type: WhatsApp media downloader  
  - Role: Fetches audio or image media content from WhatsApp servers using media IDs  
  - Inputs: Media message ID from WhatsApp JSON  
  - Outputs: Binary media data  
  - Failures: Media not found, permission issues, network timeouts  

- **HTTP Request / HTTP Request1**  
  - Type: HTTP Request  
  - Role: Retrieves media binary data from URLs provided by WhatsApp media download nodes  
  - Inputs: Media URLs  
  - Outputs: Binary data for transcription nodes  

---

#### 1.2 Message Routing and Preprocessing

**Overview:**  
After classification, merges original WhatsApp data with processing branches and prepares data for AI agents including transcription results and user inputs.

**Nodes Involved:**  
- Merge  
- Merge1  
- Merge2  
- Edit Fields / Edit Fields1 / Edit Fields2 / Edit Fields3  

**Node Details:**

- **Merge / Merge1 / Merge2**  
  - Type: Merge node  
  - Role: Synchronizes data streams, choosing data from specific branches to keep WhatsApp context intact while processing AI outputs  
  - Parameters: ‘Choose Branch’ mode  
  - Inputs: Branch data from classification and media processing  
  - Outputs: Combined JSON for next steps  
  - Edge Cases: Missing inputs, timing issues in asynchronous flows  

- **Edit Fields series**  
  - Role: Standardize inputs for AI agents by assigning key text fields from WhatsApp JSON or AI outputs  
  - Inputs: Various JSON objects  
  - Outputs: Cleaned JSON for AI processing  

---

#### 1.3 Leave Management Agent

**Overview:**  
Processes leave requests by extracting details from WhatsApp messages, verifies employee leave data from Google Sheets, decides auto-approval or escalation, sends emails to employees/department heads, and responds on WhatsApp.

**Nodes Involved:**  
- Leave Agent (Langchain agent)  
- get leaves (Google Sheets Tool)  
- update leaves (Google Sheets Tool)  
- Gmail (Gmail Tool)  
- Edit Fields3  
- Leave Agent output to WhatsApp Responder  

**Node Details:**

- **Leave Agent**  
  - Type: Langchain agent  
  - Role: Parses leave request text, checks leave balances, determines approval/escalation  
  - Tools used: Google Sheets for employee data, Gmail for emails  
  - Inputs: WhatsApp leave request text and employee name  
  - Outputs: Leave processing result and message for WhatsApp  
  - Failures: Incorrect or missing employee data, email sending failures  

- **get leaves**  
  - Type: Google Sheets Tool (Get Rows)  
  - Role: Retrieves leave balance and employee info from ‘leaves’ sheet  
  - Inputs: Employee name  
  - Outputs: Employee leave data JSON  
  - Failures: Sheet access errors, missing data  

- **update leaves**  
  - Type: Google Sheets Tool (Append or Update)  
  - Role: Updates leave records after approval or denial  
  - Inputs: Leave decision and employee data  
  - Outputs: Confirmation  
  - Failures: Write errors, concurrency issues  

- **Gmail**  
  - Type: Gmail Tool  
  - Role: Sends leave approval or escalation emails to employee and department head  
  - Inputs: Email addresses, subject, body generated by agent  
  - Outputs: Email sent confirmation  
  - Failures: SMTP/auth errors, quota limits  

- **Edit Fields3**  
  - Role: Prepares message text for Leave Agent  

---

#### 1.4 HR Chatbot Agent

**Overview:**  
Handles general HR queries from text, audio, or image messages. Uses OpenAI to transcribe media, searches company policy documents in a Supabase vector store, and composes relevant replies.

**Nodes Involved:**  
- HR Chatbot (Langchain agent)  
- OpenAI Chat Model (multiple) for transcription and response  
- Embeddings OpenAI  
- Supabase Vector Store  
- policy_vector_store (Langchain Tool)  
- WhatsApp Responder  

**Node Details:**

- **HR Chatbot**  
  - Type: Langchain agent  
  - Role: Answers FAQs using text or transcribed input, leverages vector search over policy docs, falls back to general HR knowledge if no match  
  - Inputs: Cleaned message text or transcription, policy doc embeddings  
  - Outputs: Text response for WhatsApp  
  - Failures: Transcription errors, no relevant policy found, API rate limits  

- **OpenAI Chat Models**  
  - Type: Language models for transcription and FAQ response generation  
  - Roles: Transcribe audio/image, generate answers  
  - Inputs: Binary media, text prompts  
  - Outputs: Transcriptions, text answers  

- **Embeddings OpenAI + Supabase Vector Store**  
  - Role: Generate embeddings for queries, search policy documents for closest matches  
  - Inputs: Text query  
  - Outputs: Relevant document snippets for agent response  

- **policy_vector_store**  
  - Type: Langchain vector store tool node  
  - Role: Executes vector similarity search on Supabase data  

---

#### 1.5 Attendance Management Agent

**Overview:**  
Processes GPS-based attendance marking by checking if user location is inside office boundaries, logs entry/exit times in Google Sheets, handles overtime, and responds via WhatsApp.

**Nodes Involved:**  
- Code1 (JavaScript)  
- Attendance Agent (Langchain agent)  
- getrows (Google Sheets Tool)  
- putrows (Google Sheets Tool)  
- Structured Output Parser2  
- WhatsApp Responder  

**Node Details:**

- **Code1**  
  - Type: Code node (JavaScript)  
  - Role: Geospatial check to determine if user location is inside office polygon (defined by coordinates) within 2500m radius  
  - Inputs: Location data (lat/lon) from WhatsApp location message  
  - Outputs: Boolean flag `isInsideOffice` and location details  
  - Failures: Missing location data, invalid coordinates  

- **Attendance Agent**  
  - Type: Langchain Agent  
  - Role: Based on `isInsideOffice`, logs attendance: adds entry time if none, exit time if entry exists, or notifies if already logged  
  - Uses Google Sheets to track attendance per employee per day  
  - Outputs human-readable attendance status messages  

- **getrows / putrows**  
  - Google Sheets tools to read and update attendance records  
  - Inputs: Employee name, date  
  - Outputs: Attendance records updated or created  

- **Structured Output Parser2**  
  - Parses agent output into structured JSON with message  

---

#### 1.6 Complaint/Request Email Agent

**Overview:**  
Generates and sends escalation or complaint emails to appropriate department heads, based on WhatsApp messages, using Google Sheets to fetch dept head email, and sends confirmation.

**Nodes Involved:**  
- Email Agent (Langchain agent)  
- dept head email (Google Sheets Tool)  
- Gmail2 (Gmail Tool)  
- WhatsApp Responder  

**Node Details:**

- **Email Agent**  
  - Type: Langchain agent  
  - Role: Creates professional email content from WhatsApp text and employee data, targets dept head email  
  - Inputs: WhatsApp message, employee name  
  - Outputs: Email content and recipient info  

- **dept head email**  
  - Google Sheets tool fetching email address of department head based on employee info  

- **Gmail2**  
  - Sends generated email to department head with optional CC to employee  

---

#### 1.7 Candidate Shortlisting Agent

**Overview:**  
Handles candidate shortlisting requests by extracting job description and applicant data from Google Sheets, scoring candidates using AI, writing results back to sheets, scheduling interviews, and notifying candidates.

**Nodes Involved:**  
- Shortlist Agent (Langchain agent)  
- JD tool (Google Sheets Tool)  
- applicants (Google Sheets Tool)  
- Google Sheets2 (Google Sheets)  
- Book Meeting (Langchain agent)  
- Google Calendar  
- get mail (Google Sheets Tool)  
- Gmail1 (Gmail Tool)  
- Personalize email (OpenAI)  
- WhatsApp Approval  

**Node Details:**

- **Shortlist Agent**  
  - Type: Langchain agent  
  - Role: Extracts JD and applicant info, compares CV texts, scores candidates (1-10), reasons missing skills, outputs structured data  
  - Inputs: WhatsApp message with number of candidates and position  
  - Outputs: Structured candidate evaluation data  

- **JD tool / applicants / Google Sheets2**  
  - Access job descriptions and applicants data sheets for filtering and storing results  

- **Book Meeting**  
  - Creates interview meeting 2 days later between 11am-12pm, outputs date/time info  

- **Google Calendar**  
  - Schedules meeting in Google Calendar with provided details  

- **get mail**  
  - Retrieves candidate email for notification  

- **Gmail1**  
  - Sends interview invitation email to candidate  

- **Personalize email**  
  - Generates personalized email content based on candidate data and meeting details  

- **WhatsApp Approval**  
  - Sends WhatsApp message to confirm scheduling details to user  

---

#### 1.8 Final WhatsApp Response

**Overview:**  
Collects outputs from all agents, formats final messages, and sends responses back to users on WhatsApp, supporting interactive replies.

**Nodes Involved:**  
- WhatsApp Responder  

**Node Details:**

- **WhatsApp Responder**  
  - Type: WhatsApp node  
  - Role: Sends text message responses back to user's WhatsApp number  
  - Inputs: Generated message content from agents  
  - Outputs: WhatsApp message delivery confirmation  
  - Failures: Message delivery failures, phone number errors  

---

#### 1.9 Media Handling and Transcription

**Overview:**  
Handles audio and image messages by downloading media from WhatsApp, transcribing audio or analyzing images via OpenAI, enabling HR Chatbot to process non-text input.

**Nodes Involved:**  
- download media / download media1 (WhatsApp media download)  
- HTTP Request / HTTP Request1 (fetch media content)  
- OpenAI (transcribe)  
- OpenAI1 (analyze image)  
- Edit Fields1 / Edit Fields2  

**Node Details:**

- These nodes ensure media content is fetched and converted to text for AI comprehension  
- Failures include media expiry, download errors, transcription inaccuracies  

---

### 3. Summary Table

| Node Name           | Node Type                         | Functional Role                           | Input Node(s)                  | Output Node(s)                  | Sticky Note                                                                                               |
|---------------------|----------------------------------|-----------------------------------------|-------------------------------|-------------------------------|-----------------------------------------------------------------------------------------------------------|
| WhatsApp Trigger    | WhatsApp Trigger                 | Entry point for WhatsApp messages       | —                             | Delegatory LLM, Merge, Merge1, Merge2 | LLM Message Classifier (WhatsApp Trigger) - Classifies messages into 5 categories                         |
| Delegatory LLM      | Langchain chain LLM              | Classifies incoming message intent      | WhatsApp Trigger               | Switch                        |                                                                                                           |
| Switch              | Switch                          | Routes flow based on classification     | Delegatory LLM                | Branches: Leave Agent, HR Chatbot, Attendance, Email Agent, Shortlist Agent | Switch Router (Based on LLM Output) - Routes to different agents based on classification                   |
| Verify Message Type  | Switch                          | Determines message media type            | Merge1, Merge2                | Edit Fields, download media1, download media | FAQ / Chatbot Agent - Transcribes audio/image, answers policy questions                                  |
| Edit Fields         | Set                            | Extracts message text                    | Verify Message Type            | HR Chatbot                    |                                                                                                           |
| download media1      | WhatsApp media downloader       | Downloads audio media                    | Verify Message Type            | HTTP Request                  |                                                                                                           |
| download media       | WhatsApp media downloader       | Downloads image media                    | Verify Message Type            | HTTP Request1                 |                                                                                                           |
| HTTP Request        | HTTP Request                    | Fetches audio media binary               | download media1                | OpenAI (transcribe)           |                                                                                                           |
| HTTP Request1       | HTTP Request                    | Fetches image media binary               | download media                 | OpenAI1 (analyze)             |                                                                                                           |
| OpenAI              | OpenAI (transcribe audio)       | Transcribes audio to text                 | HTTP Request                  | Edit Fields1                  |                                                                                                           |
| OpenAI1             | OpenAI (analyze image)          | Analyzes image content                   | HTTP Request1                 | Edit Fields2                  |                                                                                                           |
| Edit Fields1        | Set                            | Prepares transcription text              | OpenAI                        | HR Chatbot                    |                                                                                                           |
| Edit Fields2        | Set                            | Prepares analyzed image text             | OpenAI1                       | HR Chatbot                    |                                                                                                           |
| Merge               | Merge                          | Synchronizes location message and classification | WhatsApp Trigger, Delegatory LLM | Code1                       | Attendance Handler - Checks location inside office polygon and logs attendance                            |
| Code1               | Code (JavaScript)               | Geospatial check for attendance          | Merge                         | Attendance Agent              | Attendance Handler - Office geofence check                                                             |
| Attendance Agent     | Langchain agent                 | Marks attendance with entry/exit times   | Code1, getrows, putrows        | WhatsApp Responder            |                                                                                                           |
| getrows             | Google Sheets Tool (get rows)   | Fetches attendance records                | Attendance Agent              | Attendance Agent              |                                                                                                           |
| putrows             | Google Sheets Tool (append/update) | Updates attendance records              | Attendance Agent              | Attendance Agent              |                                                                                                           |
| Structured Output Parser2 | Output parser              | Parses attendance agent output            | Attendance Agent              | WhatsApp Responder            |                                                                                                           |
| HR Chatbot           | Langchain agent                 | Responds to HR FAQs                       | Edit Fields, Edit Fields1, Edit Fields2 | WhatsApp Responder        | FAQ / Chatbot Agent - Uses policy vector store for official answers                                     |
| Embeddings OpenAI    | OpenAI embeddings              | Generates embeddings                      | HR Chatbot                    | Supabase Vector Store         |                                                                                                           |
| Supabase Vector Store | Vector store                  | Searches policy documents                 | Embeddings OpenAI             | policy_vector_store           |                                                                                                           |
| policy_vector_store  | Langchain tool (vector store)  | Retrieves relevant policy docs            | Supabase Vector Store         | HR Chatbot                   |                                                                                                           |
| Leave Agent          | Langchain agent                 | Processes leave requests                   | Edit Fields3, get leaves, update leaves, Gmail | WhatsApp Responder | Leave Agent - Auto-approve short leaves, escalate longer leaves                                         |
| get leaves           | Google Sheets Tool              | Fetches leave data                        | Leave Agent                  | Leave Agent                  |                                                                                                           |
| update leaves        | Google Sheets Tool              | Updates leave records                     | Leave Agent                  | Leave Agent                  |                                                                                                           |
| Gmail                | Gmail Tool                     | Sends leave approval/escalation emails   | Leave Agent                  | Leave Agent                  |                                                                                                           |
| Edit Fields3         | Set                            | Prepares leave request text               | Verify Message Type           | Leave Agent                  |                                                                                                           |
| Email Agent          | Langchain agent                 | Generates complaint/escalation emails     | Switch, dept head email, Gmail2 | WhatsApp Responder          | Request/Complaint Agent - Emails dept head with complaint details                                       |
| dept head email      | Google Sheets Tool              | Gets department head email                | Email Agent                  | Email Agent                  |                                                                                                           |
| Gmail2               | Gmail Tool                     | Sends complaint/request emails            | Email Agent                  | Email Agent                  |                                                                                                           |
| Shortlist Agent      | Langchain agent                 | Shortlists candidates, scores, schedules | Switch, JD tool, applicants, Google Sheets2, Book Meeting | Google Sheets2, Book Meeting | Shortlisting Agent - Filters candidates, schedules interviews, sends notifications                      |
| JD tool              | Google Sheets Tool              | Fetches job descriptions                   | Shortlist Agent              | Shortlist Agent              |                                                                                                           |
| applicants           | Google Sheets Tool              | Fetches applicant data                     | Shortlist Agent              | Shortlist Agent              |                                                                                                           |
| Google Sheets2       | Google Sheets Tool              | Updates shortlist results                   | Shortlist Agent              | Book Meeting                 |                                                                                                           |
| Book Meeting         | Langchain agent                 | Creates interview meeting                   | Google Sheets2               | WhatsApp Approval            |                                                                                                           |
| Google Calendar      | Google Calendar Tool            | Schedules meetings                          | Book Meeting                 | Book Meeting                 |                                                                                                           |
| get mail             | Google Sheets Tool              | Gets candidate email                        | Personalize email            | Personalize email            |                                                                                                           |
| Gmail1               | Gmail Tool                     | Sends interview invitation email           | Personalize email            | Personalize email            |                                                                                                           |
| Personalize email    | OpenAI                         | Generates personalized email content       | Gmail1, get mail             | WhatsApp Responder           | Final WhatsApp Responder - Confirms actions and next steps                                             |
| WhatsApp Approval    | WhatsApp node                  | Sends meeting confirmation WhatsApp message | Book Meeting                | If                          |                                                                                                           |
| WhatsApp Responder   | WhatsApp node                  | Sends final WhatsApp replies to users       | Multiple agents              | —                           | Final WhatsApp Responder - Consolidates all agent responses                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create WhatsApp Trigger node:**  
   - Type: WhatsApp Trigger  
   - Configure webhook for message updates  
   - Set WhatsApp OAuth credentials  

2. **Add Delegatory LLM (Langchain Chain) node:**  
   - Use GPT-4 or similar model  
   - Provide prompt to classify messages strictly into categories 1-5  
   - Input: WhatsApp Trigger output JSON  
   - Output: Classification number  

3. **Add Switch node:**  
   - Route based on classification number (1 to 5)  
   - Branches for Leave Agent (1), HR Chatbot (2), Attendance (3), Complaint Email (4), Shortlisting (5)  

4. **Add Verify Message Type (Switch) node:**  
   - Check if message is text, audio, or image  
   - Route text to Edit Fields  
   - Route audio/image to media download nodes  

5. **Add WhatsApp media download nodes:**  
   - For audio and image, download media using WhatsApp API  
   - HTTP Request nodes to fetch media binary  

6. **Add OpenAI nodes:**  
   - Transcribe audio messages  
   - Analyze images  
   - Prepare transcription text for HR Chatbot agent  

7. **Add Edit Fields nodes:**  
   - Extract and normalize text fields for agents  

8. **Create Leave Agent (Langchain agent):**  
   - Connect Google Sheets ‘get leaves’ and ‘update leaves’ nodes  
   - Configure Gmail node for sending approval/escalation emails  
   - Use employee name from WhatsApp contact to fetch and update leave data  

9. **Create HR Chatbot Agent:**  
   - Connect OpenAI Chat Model nodes for transcription and answering  
   - Set up Supabase Vector Store node with policy document embeddings  
   - Use Langchain vector store tool to search policies  
   - Final output goes to WhatsApp Responder  

10. **Create Attendance Agent:**  
    - Add Code node with geofence polygon and Haversine formula  
    - Use Google Sheets ‘getrows’ and ‘putrows’ nodes for attendance logs  
    - Langchain agent to process attendance logic and format reply  
    - Structured Output Parser to standardize output  
    - Connect WhatsApp Responder  

11. **Create Complaint Email Agent:**  
    - Google Sheets node to fetch department head email based on employee  
    - Langchain agent to generate email content  
    - Gmail node to send email  
    - Connect WhatsApp Responder  

12. **Create Candidate Shortlisting Agent:**  
    - Google Sheets nodes to fetch job descriptions and applicants  
    - Langchain agent to score and reason candidate suitability  
    - Append results to Google Sheets  
    - Langchain agent to book meetings using Google Calendar node  
    - Gmail node to send personalized interview invitations  
    - WhatsApp node to confirm scheduling  

13. **Add final WhatsApp Responder node:**  
    - Send messages to user phone number  
    - Connect all agents’ outputs here for response  

14. **Credential Setup:**  
    - WhatsApp OAuth for triggers and messages  
    - OpenAI API key for GPT-4 and embeddings  
    - Google Sheets OAuth for reading/writing data  
    - Gmail OAuth for sending emails  
    - Google Calendar OAuth for scheduling meetings  
    - Supabase credentials for vector store  

15. **Test each path carefully:**  
    - Send test WhatsApp messages of each type  
    - Verify classification and routing  
    - Check leave approvals and escalations  
    - Confirm attendance marking with GPS location  
    - Test complaint emails and candidate shortlisting flows  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                      | Context or Link                                                                                                          |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------|
| LLM Message Classifier classifies WhatsApp messages strictly into 5 categories: leave request, HR FAQ, attendance, complaint, and shortlisting.                                  | Sticky Note on WhatsApp Trigger node                                                                                      |
| Switch Router routes processing based on classification number with merge nodes preserving original WhatsApp data.                                                               | Sticky Note near Switch node                                                                                               |
| Leave Agent auto-approves leave requests <2 days, escalates longer leaves to department head via email.                                                                           | Sticky Note near Leave Agent                                                                                               |
| Attendance Handler uses geofencing via code node, logs entry and exit times in Google Sheets, and supports overtime calculations.                                                | Sticky Note near Code1 and Attendance Agent                                                                                |
| Shortlisting Agent uses job description and applicant Google Sheets data to rank candidates, schedule interviews, and send personalized emails.                                   | Sticky Note near Shortlist Agent                                                                                           |
| Final WhatsApp Responder consolidates all agent responses and sends clear user messages with interactive reply options.                                                          | Sticky Note near WhatsApp Responder                                                                                        |
| FAQ/Chatbot Agent transcribes audio/image messages and searches company policy documents via vector store for accurate responses.                                               | Sticky Note near HR Chatbot                                                                                               |
| Request/Complaint Agent generates emails to escalate issues to department heads using Google Sheets data and Gmail node.                                                        | Sticky Note near Email Agent                                                                                               |
| Workflow uses multiple Google Sheets for leaves, attendance, applicants, job descriptions, and department head emails, all linked via employee names.                             | Google Sheets integration throughout                                                                                       |
| Google Calendar scheduling integrated for interview booking with human-readable date/time output.                                                                                 | Book Meeting and Google Calendar nodes                                                                                    |
| OpenAI GPT-4 models used for classification, transcription, chat responses, and email personalization.                                                                           | Multiple OpenAI nodes with GPT-4 variants                                                                                   |
| Media handling includes downloading media from WhatsApp and fetching binary data for transcription or image analysis.                                                           | Download media and HTTP Request nodes                                                                                      |

---

This comprehensive reference enables understanding, reproduction, and modification of the Automated HR Service System workflow in n8n, covering all critical nodes, logic, and integration points.
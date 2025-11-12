Save Time with AI Hiring – Automate Screening, Assessments, Alerts & Interviews

https://n8nworkflows.xyz/workflows/save-time-with-ai-hiring---automate-screening--assessments--alerts---interviews-4813


# Save Time with AI Hiring – Automate Screening, Assessments, Alerts & Interviews

### 1. Workflow Overview

This workflow automates the end-to-end hiring process using AI and integration tools, targeting HR teams and talent acquisition professionals to save time and improve candidate screening, assessment, evaluation, and communication. It processes applicant submissions, extracts and summarizes candidate profiles, evaluates semantic fit against job descriptions, manages assessment invitations and submissions, and handles interview scheduling and candidate notifications. The workflow is composed of several logical blocks:

- **1.1 Input Reception and CV Processing:** Captures candidate form submissions and uploads CVs, extracts data from resumes.
- **1.2 Applicant Profile Extraction and Storage:** Extracts structured details from resumes and adds them to Google Sheets.
- **1.3 AI-driven Profile and Job Description Summarization:** Summarizes applicant profiles and job descriptions using OpenAI.
- **1.4 Semantic Fit Evaluation:** Compares candidate profiles with job descriptions using AI to score and highlight key matches/gaps.
- **1.5 Evaluation Result Management and Approval:** Updates evaluation results, notifies Talent Acquisition (TA) for approval, and updates candidate status accordingly.
- **1.6 Assessment Invitation and Tracking:** Sends assessment forms to shortlisted candidates and tracks submissions.
- **1.7 Interview Scheduling and Status-based Notifications:** Manages interview bookings, updates statuses, and sends tailored emails based on interview outcomes.
- **1.8 Scheduled Candidate Follow-Up:** Daily checks to send assessment invitations to candidates with selected resumes.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and CV Processing

- **Overview:**  
  Captures applicant submissions through a form, uploads CVs to Google Drive, and extracts text from uploaded files for further processing.

- **Nodes Involved:**  
  - On form submission (Form Trigger)  
  - Upload CV (Google Drive)  
  - Extract from File (PDF/Text extraction)  

- **Node Details:**  

  - **On form submission**  
    - Type: Form Trigger node  
    - Role: Receives candidate application submissions through an online form webhook.  
    - Configuration: Listens to a specific webhook ID for form submission data.  
    - Inputs: External form submission HTTP request  
    - Outputs: Data containing form fields, including uploaded files  
    - Edge Cases: Missing or malformed submissions; webhook failures  
    - Version: 2.2  

  - **Upload CV**  
    - Type: Google Drive node  
    - Role: Uploads the candidate's CV file received from the form to a Google Drive folder.  
    - Configuration: Uses Google Drive OAuth2 credentials; expects file data from previous node.  
    - Inputs: File from form submission  
    - Outputs: File metadata for downstream extraction  
    - Edge Cases: Google Drive auth errors, file upload failures  
    - Version: 3  

  - **Extract from File**  
    - Type: Extract from File (PDF/Text) node  
    - Role: Extracts text content from the uploaded CV file to enable AI processing.  
    - Configuration: Default extraction settings, capable of handling PDF or text.  
    - Inputs: Uploaded file metadata  
    - Outputs: Extracted raw text of resume  
    - Edge Cases: Unsupported file format, extraction errors, very large files  
    - Version: 1  

---

#### 2.2 Applicant Profile Extraction and Storage

- **Overview:**  
  Extracts structured applicant details from the resume text using AI and stores these details into a Google Sheet for record keeping.

- **Nodes Involved:**  
  - Applicant's Details (Information Extractor AI node)  
  - Add Applicant's Details in Google Sheet (Google Sheets)  

- **Node Details:**  

  - **Applicant's Details**  
    - Type: AI Information Extractor (LangChain)  
    - Role: Processes extracted resume text to parse structured information like skills, education, job history, location.  
    - Configuration: Uses OpenAI or similar LLM credentials; customized prompts for resume data extraction.  
    - Inputs: Extracted raw resume text  
    - Outputs: Structured JSON with applicant details (skills, education, job history, personal info)  
    - Edge Cases: AI misinterpretation, incomplete data extraction, rate limits on LLM API  
    - Version: 1  

  - **Add Applicant's Details in Google Sheet**  
    - Type: Google Sheets node  
    - Role: Appends extracted structured applicant info as a new row in a centralized candidate tracking sheet.  
    - Configuration: Connected to a Google Sheets document and worksheet; maps fields like name, email, status.  
    - Inputs: Structured applicant details JSON  
    - Outputs: Confirmation of row addition  
    - Edge Cases: Sheet permission errors, quota limits, malformed data causing mapping errors  
    - Version: 4.5  

---

#### 2.3 AI-driven Profile and Job Description Summarization

- **Overview:**  
  Uses AI to generate concise summaries of both the applicant's profile and the job role description to facilitate semantic comparison.

- **Nodes Involved:**  
  - Summarize Applicant's Profile (AI Chain Summarization)  
  - Get Job Description from Google Sheets (Google Sheets)  
  - Summarize Job Role Description (AI Chain Summarization)  

- **Node Details:**  

  - **Summarize Applicant's Profile**  
    - Type: AI Chain Summarization (LangChain)  
    - Role: Summarizes the detailed extracted applicant information into a concise professional profile.  
    - Configuration: Uses LLM with summarization prompt templates.  
    - Inputs: Structured applicant details  
    - Outputs: Text summary of applicant profile  
    - Edge Cases: AI summarization quality issues, API errors  
    - Version: 2  

  - **Get Job Description from Google Sheets**  
    - Type: Google Sheets node  
    - Role: Retrieves job description and related metadata for the job applied for.  
    - Configuration: Reads specific rows matching job ID or profile from a spreadsheet.  
    - Inputs: Job profile or ID from previous node  
    - Outputs: Full job description text and metadata  
    - Edge Cases: Missing job descriptions, spreadsheet access issues  
    - Version: 4.5  

  - **Summarize Job Role Description**  
    - Type: AI Chain Summarization (LangChain)  
    - Role: Generates a concise summary of the job role to aid in semantic matching.  
    - Configuration: Uses summarization prompts with LLM.  
    - Inputs: Job description text  
    - Outputs: Text summary of job role  
    - Edge Cases: Similar to applicant summary  
    - Version: 2  

---

#### 2.4 Semantic Fit Evaluation

- **Overview:**  
  Evaluates how well the applicant fits the job description by leveraging AI to score semantic matches and identify gaps or red flags.

- **Nodes Involved:**  
  - Structured Output Parser (LangChain Output Parser)  
  - Semantic Fit & Evaluation by HR Expert (LangChain Chain LLM)  
  - Update Evaluation Results in Google Sheets (Google Sheets)  

- **Node Details:**  

  - **Structured Output Parser**  
    - Type: AI Output Parser (LangChain)  
    - Role: Parses the structured output from the semantic evaluation AI to extract meaningful evaluation data.  
    - Configuration: Configured with output schema to parse semantic fit results.  
    - Inputs: Raw output from LLM evaluation node  
    - Outputs: Structured evaluation data (scores, gaps, flags)  
    - Edge Cases: Parsing errors due to unexpected LLM output format  
    - Version: 1.2  

  - **Semantic Fit & Evaluation by HR Expert**  
    - Type: AI Chain LLM (LangChain)  
    - Role: Compares summarized applicant profile and job description, producing a semantic fit score, key matches, gaps, and overall evaluation.  
    - Configuration: Uses custom prompt templates mimicking HR expert evaluation.  
    - Inputs: Applicant and job summaries  
    - Outputs: Detailed evaluation including semantic fit score, red flags, soft skills, and final vote  
    - Edge Cases: LLM API rate limits, misinterpretation of inputs, inconsistent outputs  
    - Version: 1.5  

  - **Update Evaluation Results in Google Sheets**  
    - Type: Google Sheets node  
    - Role: Updates candidate record with semantic evaluation results and metadata.  
    - Configuration: Updates existing row with evaluation fields (scores, gaps, final vote).  
    - Inputs: Structured evaluation data  
    - Outputs: Confirmation of update  
    - Edge Cases: Sheet locking, data overwrite conflicts  
    - Version: 4.5  

---

#### 2.5 Evaluation Result Management and Approval

- **Overview:**  
  Notifies Talent Acquisition (TA) team via email about evaluation results, collects approval, and updates applicant status as shortlisted or rejected accordingly.

- **Nodes Involved:**  
  - Notify TA for Approval via Email (Email Send)  
  - Approval Check - IF Condition (If node)  
  - Update Applicant's Status as RESUME SELECTED (Google Sheets)  
  - Update Applicant's Status as REJECTED (Google Sheets)  
  - Send Shortlist Email to Candidate (Email Send)  
  - Send Rejection Email to Candidate (Email Send)  

- **Node Details:**  

  - **Notify TA for Approval via Email**  
    - Type: Email Send node  
    - Role: Sends evaluation results and approval request to TA email addresses.  
    - Configuration: SMTP credentials configured; email content includes candidate evaluation summary and approval links.  
    - Inputs: Updated evaluation data  
    - Outputs: Email delivery status  
    - Edge Cases: SMTP auth failures, email bounces  
    - Version: 2.1  

  - **Approval Check - IF Condition**  
    - Type: If node  
    - Role: Checks TA’s approval response (boolean flag) to determine next action.  
    - Configuration: Condition based on 'approved' field in email response or webhook data.  
    - Inputs: TA approval response  
    - Outputs: Branches to shortlisted or rejected paths  
    - Edge Cases: Missing or delayed response, ambiguous approval status  
    - Version: 2.2  

  - **Update Applicant's Status as RESUME SELECTED**  
    - Type: Google Sheets node  
    - Role: Updates candidate status to "RESUME SELECTED" in the tracking sheet.  
    - Configuration: Maps candidate email to row and updates status column.  
    - Inputs: Approval condition true  
    - Outputs: Confirmation of update  
    - Edge Cases: Sheet update conflicts  
    - Version: 4.5  

  - **Update Applicant's Status as REJECTED**  
    - Type: Google Sheets node  
    - Role: Updates candidate status to "REJECTED" in the tracking sheet.  
    - Configuration: Same as above but with rejected status.  
    - Inputs: Approval condition false  
    - Outputs: Confirmation of update  
    - Edge Cases: Same as above  
    - Version: 4.5  

  - **Send Shortlist Email to Candidate**  
    - Type: Email Send node  
    - Role: Sends shortlist notification to candidate’s email.  
    - Configuration: Personalized email with next steps.  
    - Inputs: Status updated to shortlisted  
    - Outputs: Email delivery status  
    - Edge Cases: SMTP errors  
    - Version: 2.1  

  - **Send Rejection Email to Candidate**  
    - Type: Email Send node  
    - Role: Sends rejection notification to candidate.  
    - Configuration: Email with polite rejection message.  
    - Inputs: Status updated to rejected  
    - Outputs: Email delivery status  
    - Edge Cases: SMTP errors  
    - Version: 2.1  

---

#### 2.6 Assessment Invitation and Tracking

- **Overview:**  
  Automatically sends assessment form links to candidates with "Resume Selected" status and tracks their submissions.

- **Nodes Involved:**  
  - Run Daily at 09:00 AM (Schedule Trigger)  
  - Fetch Records with Status "Resume Selected" (Google Sheets)  
  - Loop to Send Assessment Link to Each Candidate (Split In Batches)  
  - Get Assessment Form URL (Google Sheets)  
  - Send Assessment Submission Email (Email Send)  
  - Update Status to Assessment Sent (Google Sheets)  
  - Technical Project Manager Assessment Trigger (Typeform Trigger)  
  - Technical Support Engineer Assessment Trigger (Typeform Trigger)  
  - Update Applicant Status to Assessment Submitted (Google Sheets)  
  - Notify TA via Email for Assessment Submission (Email Send)  
  - Notify TA via Slack for Assessment Submission (Slack) [Disabled]  

- **Node Details:**  

  - **Run Daily at 09:00 AM**  
    - Type: Schedule Trigger  
    - Role: Initiates daily workflow to follow up with candidates awaiting assessments.  
    - Configuration: Fires each day at 09:00 AM server time.  
    - Outputs: Triggers next node  
    - Version: 1.2  

  - **Fetch Records with Status "Resume Selected"**  
    - Type: Google Sheets node  
    - Role: Retrieves all candidates marked as "Resume Selected" to identify who needs assessment invites.  
    - Configuration: Uses sheet filters for status column.  
    - Outputs: List of candidate records  
    - Edge Cases: Large dataset pagination, API limits  
    - Version: 4.5  

  - **Loop to Send Assessment Link to Each Candidate**  
    - Type: Split In Batches node  
    - Role: Processes candidates one by one (or in small batches) to send assessment links individually.  
    - Configuration: Batch size set (default 1 or configurable).  
    - Inputs: List of candidates  
    - Outputs: Single candidate data per batch  
    - Version: 3  

  - **Get Assessment Form URL**  
    - Type: Google Sheets node  
    - Role: Retrieves the correct assessment form URL based on the candidate’s applied job or role.  
    - Configuration: Uses job ID or profile to fetch corresponding form link.  
    - Inputs: Candidate job profile  
    - Outputs: Assessment form URL  
    - Edge Cases: Missing form URLs, incorrect mapping  
    - Version: 4.5  

  - **Send Assessment Submission Email**  
    - Type: Email Send node  
    - Role: Sends the assessment form link to the candidate’s email with instructions.  
    - Configuration: Uses SMTP or transactional email service; personalized content.  
    - Inputs: Candidate email, assessment form URL  
    - Outputs: Email send status  
    - Edge Cases: Invalid email addresses, SMTP issues  
    - Version: 2.1  

  - **Update Status to Assessment Sent**  
    - Type: Google Sheets node  
    - Role: Updates candidate status to "Assessment Sent" and records timestamp.  
    - Inputs: Candidate record after email sent  
    - Outputs: Confirmation  
    - Version: 4.5  

  - **Technical Project Manager Assessment Trigger / Technical Support Engineer Assessment Trigger**  
    - Type: Typeform Trigger nodes  
    - Role: Triggered when candidate submits assessment form for respective roles.  
    - Configuration: Webhook triggers linked to Typeform forms; listens for submissions.  
    - Inputs: Assessment submission data  
    - Outputs: Trigger downstream processing  
    - Edge Cases: Typeform webhook misconfigurations, submission delays  
    - Version: 1.1  

  - **Update Applicant Status to Assessment Submitted**  
    - Type: Google Sheets node  
    - Role: Marks candidate status as "Assessment Submitted" upon receiving form completion.  
    - Inputs: Submission trigger data  
    - Outputs: Confirmation  
    - Version: 4.5  

  - **Notify TA via Email for Assessment Submission**  
    - Type: Email Send node  
    - Role: Notifies TA that candidate has completed assessment.  
    - Inputs: Candidate info on submission  
    - Outputs: Email sent confirmation  
    - Version: 2.1  

  - **Notify TA via Slack for Assessment Submission**  
    - Type: Slack node (disabled)  
    - Role: Intended to send Slack notification to TA channel on assessment submission (currently disabled).  
    - Edge Cases: Slack API errors, disabled node status  
    - Version: 2.3  

---

#### 2.7 Interview Scheduling and Status-based Notifications

- **Overview:**  
  Manages interview bookings via Calendly, updates candidate status, and sends customized emails based on interview outcomes or status changes.

- **Nodes Involved:**  
  - Trigger when Interview booked by applicant in Calendly (Calendly Trigger)  
  - Update Status to Interview Booked (Google Sheets)  
  - Get Triggered when Applicant Status Update in Google Sheet (Google Sheets Trigger)  
  - Route actions based on Status (Switch node)  
  - Send Interview Invite Email (Email Send)  
  - Send Assessment Failed Email (Email Send)  
  - Send Interview Cancelled Email (Email Send)  
  - Interview Reschedule Invite Email (Email Send)  
  - Send Interview Passed/Shortlisted Email (Email Send)  
  - Send Interview Failed Email (Email Send)  

- **Node Details:**  

  - **Trigger when Interview booked by applicant in Calendly**  
    - Type: Calendly Trigger  
    - Role: Listens to interview bookings made by candidates via Calendly scheduling tool.  
    - Configuration: Configured with Calendly webhook credentials.  
    - Outputs: Interview booking data  
    - Edge Cases: Webhook delivery failures, Calendly API limits  
    - Version: 1  

  - **Update Status to Interview Booked**  
    - Type: Google Sheets node  
    - Role: Updates candidate status in sheet to "Interview Booked" upon Calendly trigger.  
    - Inputs: Booking data linking candidate email  
    - Outputs: Confirmation of update  
    - Version: 4.6  

  - **Get Triggered when Applicant Status Update in Google Sheet**  
    - Type: Google Sheets Trigger  
    - Role: Watches for any status changes in candidate sheet to route next actions.  
    - Configuration: Trigger on row update events filtered by status column.  
    - Outputs: Changed row data  
    - Version: 1  

  - **Route actions based on Status**  
    - Type: Switch node  
    - Role: Routes workflow based on candidate status (e.g., Interview Invited, Assessment Failed, Interview Cancelled, etc.)  
    - Configuration: Multiple condition branches for each status string  
    - Inputs: Updated status from sheet trigger  
    - Outputs: Branches to appropriate email notification nodes or no-op  
    - Version: 3.2  

  - **Send Interview Invite Email**  
    - Type: Email Send node  
    - Role: Sends interview invitation email to candidate.  
    - Inputs: Candidate contact info  
    - Outputs: Email confirmation  
    - Version: 2.1  

  - **Send Assessment Failed Email**  
    - Type: Email Send node  
    - Role: Notifies candidate of assessment failure.  
    - Version: 2.1  

  - **Send Interview Cancelled Email**  
    - Type: Email Send node  
    - Role: Notifies candidate about interview cancellation.  
    - Version: 2.1  

  - **Interview Reschedule Invite Email**  
    - Type: Email Send node  
    - Role: Sends rescheduling options to candidate.  
    - Version: 2.1  

  - **Send Interview Passed/Shortlisted Email**  
    - Type: Email Send node  
    - Role: Congratulates candidate on passing the interview stage.  
    - Version: 2.1  

  - **Send Interview Failed Email**  
    - Type: Email Send node  
    - Role: Sends rejection notification post interview failure.  
    - Version: 2.1  

---

#### 2.8 Scheduled Candidate Follow-Up

- **Overview:**  
  Runs daily to check for candidates with pending assessments and sends them assessment invitations if not yet sent.

- **Nodes Involved:**  
  - Run Daily at 09:00 AM (Schedule Trigger)  
  - Fetch Records with Status "Resume Selected" (Google Sheets)  
  - Loop to Send Assessment Link to Each Candidate (Split In Batches)  
  - Get Assessment Form URL (Google Sheets)  
  - Send Assessment Submission Email (Email Send)  
  - Update Status to Assessment Sent (Google Sheets)  

- **Node Details:**  
  These nodes have been described above in 2.6 and form a daily follow-up sub-block.

---

### 3. Summary Table

| Node Name                                    | Node Type                        | Functional Role                                   | Input Node(s)                             | Output Node(s)                            | Sticky Note                         |
|----------------------------------------------|---------------------------------|-------------------------------------------------|-----------------------------------------|------------------------------------------|-----------------------------------|
| On form submission                            | Form Trigger                    | Receives application submissions                 | -                                       | Upload CV, Extract from File              |                                   |
| Upload CV                                    | Google Drive                   | Uploads CV to Google Drive                         | On form submission                      | Extract from File                         |                                   |
| Extract from File                            | Extract from File               | Extracts text from CV file                         | Upload CV                              | Add Applicant's Details in Google Sheet  |                                   |
| Add Applicant's Details in Google Sheet      | Google Sheets                   | Adds structured applicant data to sheet           | Extract from File                      | Applicant's Details                       |                                   |
| Applicant's Details                          | AI Information Extractor        | Extracts structured applicant info via AI         | Add Applicant's Details in Google Sheet | Summarize Applicant's Profile             |                                   |
| Summarize Applicant's Profile                | AI Chain Summarization          | Generates summary of applicant profile             | Applicant's Details                   | Get Job Description from Google Sheets    |                                   |
| Get Job Description from Google Sheets        | Google Sheets                   | Retrieves job description for position             | Summarize Applicant's Profile          | Summarize Job Role Description            |                                   |
| Summarize Job Role Description                | AI Chain Summarization          | Summarizes job role description                     | Get Job Description from Google Sheets | Semantic Fit & Evaluation by HR Expert    |                                   |
| Semantic Fit & Evaluation by HR Expert        | AI Chain LLM                   | Evaluates semantic fit and scores candidate        | Summarize Applicant's Profile, Summarize Job Role Description | Structured Output Parser                  |                                   |
| Structured Output Parser                      | AI Output Parser               | Parses AI evaluation output into structured data   | Semantic Fit & Evaluation by HR Expert | Update Evaluation Results in Google Sheets |                                   |
| Update Evaluation Results in Google Sheets    | Google Sheets                   | Updates candidate record with evaluation results   | Structured Output Parser              | Notify TA for Approval via Email          |                                   |
| Notify TA for Approval via Email              | Email Send                     | Sends approval request to TA                         | Update Evaluation Results in Google Sheets | Approval Check - IF Condition             |                                   |
| Approval Check - IF Condition                  | If                             | Routes workflow based on TA approval decision       | Notify TA for Approval via Email      | Update Applicant's Status as RESUME SELECTED / Update Applicant's Status as REJECTED |                                   |
| Update Applicant's Status as RESUME SELECTED | Google Sheets                   | Updates status to shortlisted                        | Approval Check - IF Condition          | Send Shortlist Email to Candidate          |                                   |
| Send Shortlist Email to Candidate              | Email Send                     | Sends shortlist notification to candidate           | Update Applicant's Status as RESUME SELECTED | -                                        |                                   |
| Update Applicant's Status as REJECTED          | Google Sheets                   | Updates status to rejected                           | Approval Check - IF Condition          | Send Rejection Email to Candidate          |                                   |
| Send Rejection Email to Candidate               | Email Send                     | Sends rejection notification                         | Update Applicant's Status as REJECTED | -                                        |                                   |
| Run Daily at 09:00 AM                         | Schedule Trigger               | Triggers daily assessment invitation process        | -                                       | Fetch Records with Status "Resume Selected" |                                   |
| Fetch Records with Status "Resume Selected"    | Google Sheets                   | Fetches candidates with selected resumes            | Run Daily at 09:00 AM                 | Loop to Send Assessment Link to Each Candidate |                                   |
| Loop to Send Assessment Link to Each Candidate | Split In Batches               | Processes candidates individually for assessment emails | Fetch Records with Status "Resume Selected" | Get Assessment Form URL, No Operation     |                                   |
| No Operation, do nothing                      | NoOp                           | Placeholder for no action                            | Loop to Send Assessment Link to Each Candidate | Get Assessment Form URL                 |                                   |
| Get Assessment Form URL                       | Google Sheets                   | Retrieves assessment form URL based on job profile | Loop to Send Assessment Link to Each Candidate | Send Assessment Submission Email          |                                   |
| Send Assessment Submission Email              | Email Send                     | Sends assessment form link to candidate              | Get Assessment Form URL               | Update Status to Assessment Sent            |                                   |
| Update Status to Assessment Sent               | Google Sheets                   | Updates status to "Assessment Sent"                  | Send Assessment Submission Email      | Loop to Send Assessment Link to Each Candidate |                                   |
| Technical Project Manager Assessment Trigger   | Typeform Trigger               | Triggers on assessment submission for Project Manager | -                                   | Update Applicant Status to Assessment Submitted |                                   |
| Technical Support Engineer Assessment Trigger  | Typeform Trigger               | Triggers on assessment submission for Support Engineer | -                                   | Update Applicant Status to Assessment Submitted |                                   |
| Update Applicant Status to Assessment Submitted | Google Sheets                   | Updates status to "Assessment Submitted"             | Technical Project Manager / Support Engineer Assessment Trigger | Notify TA via Email for Assessment Submission |                                   |
| Notify TA via Email for Assessment Submission  | Email Send                     | Notifies TA about assessment submission              | Update Applicant Status to Assessment Submitted | -                                        |                                   |
| Notify TA via Slack for Assessment Submission  | Slack (disabled)               | Intended to notify TA via Slack (currently disabled) | Update Applicant Status to Assessment Submitted | -                                        | Node is disabled                 |
| Trigger when Interview booked by applicant in calendly | Calendly Trigger              | Triggers on interview booking by candidate            | -                                       | Update Status to Interview Booked           |                                   |
| Update Status to Interview Booked              | Google Sheets                   | Updates status to "Interview Booked"                  | Trigger when Interview booked by applicant in calendly | -                                        |                                   |
| Get Triggered when Applicant Status Update in Google Sheet | Google Sheets Trigger          | Watches for status updates in candidate sheet          | -                                       | Route actions based on Status               |                                   |
| Route actions based on Status                   | Switch                         | Routes workflow based on candidate status              | Get Triggered when Applicant Status Update in Google Sheet | Multiple email send nodes                   |                                   |
| Send Interview Invite Email                     | Email Send                     | Sends interview invitation email                       | Route actions based on Status         | -                                        |                                   |
| Send Assessment Failed Email                     | Email Send                     | Sends assessment failure email                          | Route actions based on Status         | -                                        |                                   |
| Send Interview Cancelled Email                   | Email Send                     | Sends interview cancellation email                      | Route actions based on Status         | -                                        |                                   |
| Interview Reschedule Invite Email                 | Email Send                     | Sends reschedule invitation email                       | Route actions based on Status         | -                                        |                                   |
| Send Interview Passed/Shortlisted Email          | Email Send                     | Sends interview passed notification                      | Route actions based on Status         | -                                        |                                   |
| Send Interview Failed Email                       | Email Send                     | Sends interview failed notification                      | Route actions based on Status         | -                                        |                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node "On form submission"**  
   - Type: Form Trigger  
   - Configure webhook to capture applicant submissions including CV uploads.  
   - Save and activate webhook.

2. **Add Google Drive Node "Upload CV"**  
   - Connect input from "On form submission" node.  
   - Configure with Google Drive OAuth2 credentials.  
   - Set to upload received CV file from form data.

3. **Add Extract From File Node "Extract from File"**  
   - Connect input from "Upload CV".  
   - Default extraction settings; ensure support for PDF and text files.

4. **Add Google Sheets Node "Add Applicant's Details in Google Sheet"**  
   - Connect input from "Extract from File".  
   - Configure to write to candidate tracking Google Sheet.  
   - Map fields: Name, Email, Phone, Status ("CV SUBMITTED"), Job Profile, Last Updated Date.

5. **Add AI Information Extractor Node "Applicant's Details"**  
   - Connect input from "Add Applicant's Details in Google Sheet".  
   - Configure OpenAI credentials with prompts to extract structured resume data (skills, education, job history, location).

6. **Add AI Chain Summarization Node "Summarize Applicant's Profile"**  
   - Connect input from "Applicant's Details".  
   - Configure prompt to summarize structured applicant info into a concise profile.

7. **Add Google Sheets Node "Get Job Description from Google Sheets"**  
   - Connect input from "Summarize Applicant's Profile".  
   - Configure to read job description matching candidate's job profile.

8. **Add AI Chain Summarization Node "Summarize Job Role Description"**  
   - Connect input from "Get Job Description from Google Sheets".  
   - Configure prompt to summarize job description.

9. **Add AI Chain LLM Node "Semantic Fit & Evaluation by HR Expert"**  
   - Connect inputs from "Summarize Applicant's Profile" and "Summarize Job Role Description".  
   - Configure prompt to compare and score semantic fit, identify gaps, red flags, soft skills, and provide evaluation vote.

10. **Add AI Output Parser Node "Structured Output Parser"**  
    - Connect input from "Semantic Fit & Evaluation by HR Expert".  
    - Configure parser schema to extract structured evaluation output.

11. **Add Google Sheets Node "Update Evaluation Results in Google Sheets"**  
    - Connect input from "Structured Output Parser".  
    - Configure to update candidate row with evaluation results and semantic fit data.

12. **Add Email Send Node "Notify TA for Approval via Email"**  
    - Connect input from "Update Evaluation Results in Google Sheets".  
    - Configure SMTP credentials.  
    - Compose email containing evaluation summary and approval request.

13. **Add If Node "Approval Check - IF Condition"**  
    - Connect input from "Notify TA for Approval via Email".  
    - Configure condition to check TA approval response boolean.

14. **Add Google Sheets Node "Update Applicant's Status as RESUME SELECTED"**  
    - Connect input from true branch of approval check.  
    - Configure to update candidate status to "RESUME SELECTED".

15. **Add Google Sheets Node "Update Applicant's Status as REJECTED"**  
    - Connect input from false branch of approval check.  
    - Configure to update candidate status to "REJECTED".

16. **Add Email Send Node "Send Shortlist Email to Candidate"**  
    - Connect input from "Update Applicant's Status as RESUME SELECTED".  
    - Configure to send shortlist notification email.

17. **Add Email Send Node "Send Rejection Email to Candidate"**  
    - Connect input from "Update Applicant's Status as REJECTED".  
    - Configure to send rejection notification email.

18. **Add Schedule Trigger Node "Run Daily at 09:00 AM"**  
    - Configure to trigger daily at 09:00 AM.

19. **Add Google Sheets Node "Fetch Records with Status 'Resume Selected'"**  
    - Connect input from schedule trigger.  
    - Configure to fetch all candidates with status "RESUME SELECTED".

20. **Add Split in Batches Node "Loop to Send Assessment Link to Each Candidate"**  
    - Connect input from previous node.  
    - Configure batch size 1 for sequential processing.

21. **Add Google Sheets Node "Get Assessment Form URL"**  
    - Connect input from split batches node.  
    - Configure to retrieve appropriate assessment URL based on candidate job profile.

22. **Add Email Send Node "Send Assessment Submission Email"**  
    - Connect input from "Get Assessment Form URL".  
    - Configure to send assessment form link to candidate email.

23. **Add Google Sheets Node "Update Status to Assessment Sent"**  
    - Connect input from "Send Assessment Submission Email".  
    - Update candidate status to "ASSESSMENT SENT".

24. **Add Typeform Trigger Nodes for Assessments**  
    - Create triggers for each assessment form (Technical Project Manager, Technical Support Engineer).  
    - Configure with respective Typeform webhook URLs.

25. **Add Google Sheets Node "Update Applicant Status to Assessment Submitted"**  
    - Connect input from Typeform trigger nodes.  
    - Update candidate status to "ASSESSMENT SUBMITTED".

26. **Add Email Send Node "Notify TA via Email for Assessment Submission"**  
    - Connect input from status update node above.  
    - Notify TA about assessment submission.

27. **Add Calendly Trigger Node "Trigger when Interview booked by applicant in calendly"**  
    - Configure webhook to listen to Calendly interview bookings.

28. **Add Google Sheets Node "Update Status to Interview Booked"**  
    - Connect input from Calendly trigger.  
    - Update candidate status to "INTERVIEW BOOKED".

29. **Add Google Sheets Trigger Node "Get Triggered when Applicant Status Update in Google Sheet"**  
    - Monitor status changes in candidate sheet.

30. **Add Switch Node "Route actions based on Status"**  
    - Connect input from sheet trigger.  
    - Configure branches for statuses such as "INTERVIEW INVITED", "ASSESSMENT FAILED", "INTERVIEW CANCELLED", "INTERVIEW PASSED", etc.

31. **Add Email Send Nodes for Various Interview Outcomes**  
    - Add email nodes for sending interview invites, assessment failure notices, interview cancellations, reschedule invitations, pass/shortlist notifications, and interview failure notices.  
    - Connect these to respective branches of the switch node.

32. **Activate and Test Workflow**  
    - Ensure all credentials (OpenAI, Google OAuth2, SMTP, Calendly, Typeform) are configured properly.  
    - Test each trigger and path with sample data to confirm expected flow and error handling.

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                                                                     |
|-----------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This workflow integrates multiple external services including Google Sheets, Drive, OpenAI, Calendly, Typeform, and email SMTP services. | Requires corresponding API credentials and webhook configurations.                                |
| Slack notification node is present but currently disabled, can be enabled to notify TA team on assessment submissions. | Slack integration is optional and requires Slack app credentials and channel configuration.      |
| The semantic fit evaluation uses AI prompts designed to mimic HR expert review, enhancing recruitment quality. | Custom prompts and LLM model choice are critical for evaluation accuracy.                          |
| For best performance, ensure Google Sheets have proper column headers matching node mappings. | Column names and sheet structure should be consistent with workflow expectations.                  |
| Workflow handles candidate status updates extensively to trigger subsequent actions automatically. | Status values are case-sensitive and should be standardized across the sheet and workflow.        |
| Email templates should be configured with professional, clear messaging tailored to each candidate stage. | SMTP or transactional email service credentials required; consider using services with high deliverability. |

---

**Disclaimer:**  
The provided text and workflow are derived exclusively from an automated n8n workflow. This processing strictly complies with content policies and contains no illegal, offensive, or protected materials. All data manipulated is legal and public.
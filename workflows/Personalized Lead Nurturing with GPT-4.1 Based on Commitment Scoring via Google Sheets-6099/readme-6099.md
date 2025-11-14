Personalized Lead Nurturing with GPT-4.1 Based on Commitment Scoring via Google Sheets

https://n8nworkflows.xyz/workflows/personalized-lead-nurturing-with-gpt-4-1-based-on-commitment-scoring-via-google-sheets-6099


# Personalized Lead Nurturing with GPT-4.1 Based on Commitment Scoring via Google Sheets

### 1. Workflow Overview

This workflow automates personalized lead nurturing emails based on a commitment score collected via Google Sheets. It targets boutique agencies or personal branding firms seeking to engage leads with customized messaging that reflects their self-reported commitment level to their goals.

**Logical Blocks:**

- **1.1 Input Reception**  
  Captures new lead information and commitment scores from a Google Sheet in real-time using a Google Sheets Trigger.

- **1.2 Commitment Scoring Decision**  
  Evaluates whether the lead’s commitment score is greater than or equal to 8 to determine the email tone and content.

- **1.3 AI-Powered Email Crafting**  
  Generates personalized email drafts using GPT-4.1 with distinct templates for "warmer" (high commitment) and "colder" (lower commitment) leads.

- **1.4 Email Dispatching**  
  Sends the crafted emails via Gmail using OAuth2 authentication.

- **1.5 Data Logging**  
  Appends lead and email engagement details to a separate Google Sheet for record-keeping and further CRM use.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Monitors a specified Google Sheet for newly added rows, triggering the workflow each time a lead submits data.

- **Nodes Involved:**  
  - Google Sheets Trigger

- **Node Details:**

  **Google Sheets Trigger**  
  - Type: Trigger node specific to Google Sheets.  
  - Configuration: Triggers on new rows added to the sheet with ID `1bDfwrU6m7UmR9b3pYKRh3tkM9Y-g4CAMo7Q_x1Pbx-k` and sheet gid=0. Polls every minute.  
  - Key Expressions: Extracts data fields such as "First name", "Email", "1-10 how committed are you to your goals", "Industry", and goal descriptions from the new row JSON.  
  - Input: External event (new row).  
  - Output: Passes lead data as JSON downstream.  
  - Edge Cases:  
    - Potential Google API quota limits or auth token expiry.  
    - Data schema changes may cause missing fields or expression failures.  
  - Credential: Google Sheets OAuth2 (configured).  

#### 2.2 Commitment Scoring Decision

- **Overview:**  
  Uses an If node to branch the workflow based on whether the lead’s commitment score is 8 or above.

- **Nodes Involved:**  
  - Commitment ≥ 8?

- **Node Details:**

  **Commitment ≥ 8?**  
  - Type: If node.  
  - Configuration: Checks if `1-10 how commited are you to your goals` field value ≥ 8 (numeric comparison).  
  - Input: Lead data from Google Sheets Trigger.  
  - Output: Two branches:  
    - True branch (≥8) → warmer email crafting  
    - False branch (<8) → colder email crafting  
  - Edge Cases:  
    - Missing or non-numeric commitment score causes expression failure.  
    - Case sensitivity and strict type validation used to avoid false positives.  

#### 2.3 AI-Powered Email Crafting

- **Overview:**  
  Uses OpenAI GPT-4.1 to generate personalized email drafts based on lead data, with distinct content and tone depending on commitment level.

- **Nodes Involved:**  
  - crafting warmer email  
  - crafting colder email

- **Node Details:**

  **crafting warmer email**  
  - Type: Langchain OpenAI node.  
  - Configuration:  
    - Model: GPT-4.1  
    - Temperature: 0.7 (balanced creativity)  
    - System prompt: Boutique agency’s automated concierge writing concise, uplifting emails.  
    - User prompt: Personalized first-touch email including lead’s first name, commitment score, industry, restatement of goals, invitation to a discovery call, mention of flagship offers, with specific tone instructions (aspirational, confident, warm).  
    - Output: JSON array with "subject" and "message" fields.  
  - Input: Lead data from Commitment ≥ 8? node.  
  - Output: Email subject and HTML-formatted message.  
  - Edge Cases:  
    - API rate limits or key expiry.  
    - Output JSON parsing errors if AI response deviates from expected format.  

  **crafting colder email**  
  - Type: Langchain OpenAI node.  
  - Configuration:  
    - Model: GPT-4.1  
    - Temperature: 0.7  
    - System prompt: Same as warmer email node.  
    - User prompt: Gentle nudge email acknowledging lower commitment score, thanking for sharing goals, providing ROI bullet points on personal branding, asking reflective question, offering a no-pressure Q&A session, concise subject line sparking curiosity, no em-dashes.  
    - Output: JSON array with "subject" and "message".  
  - Input: Lead data from Commitment < 8 branch.  
  - Output: Email subject and HTML message.  
  - Edge Cases: Same as warmer email node.  

#### 2.4 Email Dispatching

- **Overview:**  
  Sends the generated emails to the lead’s email address via Gmail, using OAuth2 authentication.

- **Nodes Involved:**  
  - Send warmer email  
  - Send colder email

- **Node Details:**

  **Send warmer email**  
  - Type: Gmail node (send email).  
  - Configuration:  
    - Recipient: Lead’s Email from Commitment ≥ 8? node JSON.  
    - Subject and message: Taken from the output of "crafting warmer email" node.  
    - Option: Attribution disabled (no appended signature).  
  - Input: Email draft from crafting warmer email.  
  - Output: Success confirmation flow to next node.  
  - Credential: Gmail OAuth2 configured.  
  - Edge Cases:  
    - Gmail API quota exceeded or auth token invalid.  
    - Invalid email formats could cause send failures.  

  **Send colder email**  
  - Type: Gmail node (send email).  
  - Configuration: Similar to Send warmer email, but uses output from crafting colder email node and email address from Google Sheets Trigger directly.  
  - Edge Cases: Same as above.  

#### 2.5 Data Logging

- **Overview:**  
  Appends lead details along with commitment score and goals to a separate Google Sheet for tracking and CRM integration.

- **Nodes Involved:**  
  - Append row in sheet

- **Node Details:**

  **Append row in sheet**  
  - Type: Google Sheets node (append operation).  
  - Configuration:  
    - Document ID: `1YHA1nfFamFhx2Hid9IYO14Op_P0NgXdPiGsWgD3t0Do`  
    - Sheet: gid=0  
    - Columns mapped:  
      - name: concatenation of "First name" and "Last name" from lead data  
      - email, phone, title, industry, commitment (with "/10" suffix), and three goals fields mapped from lead JSON  
  - Input: From both Send warmer email and Send colder email nodes (merged path).  
  - Output: None (terminal)  
  - Credential: Google Sheets OAuth2 configured.  
  - Edge Cases:  
    - Permissions or quota limits on Google Sheets API.  
    - Data mismatch or schema changes causing append errors.  

---

### 3. Summary Table

| Node Name             | Node Type                  | Functional Role                     | Input Node(s)          | Output Node(s)           | Sticky Note                                                                                                                                                         |
|-----------------------|----------------------------|-----------------------------------|------------------------|--------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Google Sheets Trigger  | Google Sheets Trigger       | Input reception from Google Sheets| (Trigger)              | Commitment ≥ 8?          | ### 1. Google sheets trigger and at the end can be replaced with whatever CRM                                                                                     |
| Commitment ≥ 8?       | If                         | Decision on commitment score       | Google Sheets Trigger  | crafting warmer email, crafting colder email | ### 2. If commitment is >/= to 8 then a warm email is crafted and a colder is crafted if commitment is less than 8                                               |
| crafting warmer email  | Langchain OpenAI (GPT-4.1) | Generate warm personalized email  | Commitment ≥ 8? (true) | Send warmer email        |                                                                                                                                                                   |
| crafting colder email  | Langchain OpenAI (GPT-4.1) | Generate cold personalized email  | Commitment ≥ 8? (false)| Send colder email        |                                                                                                                                                                   |
| Send warmer email      | Gmail                      | Send warm email                   | crafting warmer email  | Append row in sheet      | ### 3. Email drafts are made and CRM/Sheets is updated                                                                                                           |
| Send colder email      | Gmail                      | Send cold email                   | crafting colder email  | Append row in sheet      | ### 3. Email drafts are made and CRM/Sheets is updated                                                                                                           |
| Append row in sheet    | Google Sheets               | Log lead data and email details   | Send warmer email, Send colder email | (terminal)               |                                                                                                                                                                   |
| Sticky Note            | Sticky Note                 | Workflow overview and instructions|                        |                          | ## Workflow Overview  \n\nThis workflow uses a webhook to trigger the creation of personalized onboarding emails.\n\n### Instructions\n\n1. Modify the **OpenAI** nodes.\n2. Modify the **Gmail** nodes.\n3. Modify the **Google Sheets** nodes. |
| Sticky Note1           | Sticky Note                 | Comment on input trigger          |                        |                          | ### 1. Google sheets trigger and at the end can be replaced with whatever CRM                                                                                     |
| Sticky Note2           | Sticky Note                 | Comment on commitment decision    |                        |                          | ### 2. If commitment is >/= to 8 then a warm email is crafted and a colder is crafted if commitment is less than 8                                               |
| Sticky Note3           | Sticky Note                 | Comment on email generation & logging |                    |                          | ### 3. Email drafts are made and CRM/Sheets is updated                                                                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Google Sheets Trigger node:**  
   - Set event to "rowAdded" on the sheet with ID `1bDfwrU6m7UmR9b3pYKRh3tkM9Y-g4CAMo7Q_x1Pbx-k`, sheet gid=0.  
   - Poll every 1 minute.  
   - Configure OAuth2 credentials for Google Sheets.  

2. **Add an If node named "Commitment ≥ 8?":**  
   - Condition: Check if the field `1-10 how commited are you to your goals` (from trigger JSON) is a number ≥ 8.  
   - Use strict type validation and case-sensitive matching.  
   - Connect Google Sheets Trigger node output to this node.  

3. **Add an OpenAI GPT-4.1 node named "crafting warmer email":**  
   - Use Langchain OpenAI node or n8n OpenAI node with GPT-4.1 model.  
   - Temperature: 0.7.  
   - System prompt: "You are a boutique Agency’s highly intelligent automated concierge. Write concise, uplifting emails that sound like a boutique personal-branding firm."  
   - User prompt: Personalized first-touch email with variables: first name, commitment score, industry, goals, invitation link, flagship offers, tone aspirational/confident/warm. Include subject ≤50 chars, no em-dashes, output JSON array with subject and message fields.  
   - Connect the true output of "Commitment ≥ 8?" to this node.  
   - Set OpenAI credentials accordingly.  

4. **Add an OpenAI GPT-4.1 node named "crafting colder email":**  
   - Same model and temperature as above.  
   - System prompt identical.  
   - User prompt: Gentle nudge email with encouragement, thanks, ROI bullet points, reflective question, 15-min Q&A session link, warm sign-off, concise curiosity-sparking subject, no em-dashes, under 200 words. Output JSON array with subject and message.  
   - Connect false output of "Commitment ≥ 8?" to this node.  
   - Use OpenAI credentials.  

5. **Add two Gmail nodes named "Send warmer email" and "Send colder email":**  
   - For "Send warmer email":  
     - SendTo: `={{ $('Commitment ≥ 8?').item.json.Email }}`  
     - Subject & message: from "crafting warmer email" output JSON fields (`subject` and `message`)  
     - Disable appended attribution.  
     - Connect "crafting warmer email" output to this node.  
   - For "Send colder email":  
     - SendTo: `={{ $('Google Sheets Trigger').item.json.Email }}`  
     - Subject & message: from "crafting colder email" output JSON fields.  
     - Disable appended attribution.  
     - Connect "crafting colder email" output to this node.  
   - Configure Gmail OAuth2 credentials for both.  

6. **Add a Google Sheets node named "Append row in sheet":**  
   - Operation: Append.  
   - Document ID: `1YHA1nfFamFhx2Hid9IYO14Op_P0NgXdPiGsWgD3t0Do`  
   - Sheet gid=0.  
   - Map columns:  
     - name: concatenate `First name` + `Last name` from the "Commitment ≥ 8?" node JSON.  
     - email, phone, title, industry, three goals, commitment score with "/10" appended similarly from "Commitment ≥ 8?" node JSON.  
   - Connect outputs of both "Send warmer email" and "Send colder email" nodes to this node (merge branches).  
   - Use Google Sheets OAuth2 credentials.  

7. **Optional Sticky Notes:**  
   - Create notes to document overview, instructions, and logical blocks as in the original workflow for clarity and maintainability.  

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                                                                         |
|------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Workflow uses GPT-4.1 for natural language generation of personalized emails, requiring OpenAI API key. | OpenAI API documentation: https://platform.openai.com/docs/api-reference                             |
| Gmail nodes rely on OAuth2 authentication; ensure tokens are refreshed to avoid sending failures.     | Gmail OAuth2 setup guide: https://developers.google.com/identity/protocols/oauth2                      |
| Google Sheets Trigger and Append nodes depend on sheet structure; changing sheets or columns requires updating expressions. | Google Sheets API reference: https://developers.google.com/sheets/api/reference/rest                  |
| Email content avoids em-dashes due to formatting restrictions in some email clients.                  | Best practices for email compatibility: https://www.campaignmonitor.com/resources/guides/email-design/|
| Calendly link used in emails: https://calendly.com/brandied/discovery                                    | Booking link embedded in email invitations.                                                           |

---

_Disclaimer: The text provided is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly adheres to current content policies and contains no illegal, offensive, or protected material. All manipulated data is legal and publicly accessible._
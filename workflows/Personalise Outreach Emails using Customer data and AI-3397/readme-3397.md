Personalise Outreach Emails using Customer data and AI

https://n8nworkflows.xyz/workflows/personalise-outreach-emails-using-customer-data-and-ai-3397


# Personalise Outreach Emails using Customer data and AI

### 1. Workflow Overview

This workflow automates the creation of personalized outreach emails by leveraging existing customer email correspondence and AI analysis. It targets sales or marketing professionals who want to improve email engagement by tailoring messages based on customer communication styles and preferences extracted from their past emails.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Manual trigger to start the workflow.
- **1.2 Customer Retrieval:** Pull targeted customers from HubSpot CRM.
- **1.3 Loop Through Each Customer:** Iterate over each customer to process them individually.
- **1.4 Retrieve Customer Emails:** Fetch all emails received from each customer via Gmail.
- **1.5 Build Customer Persona:** Use AI to analyze the customer's emails and extract a detailed persona including decision-making style, communication preferences, and pain points.
- **1.6 Generate Personalized Sales Email:** Use the generated persona and product details to create a tailored outreach email via AI.
- **1.7 Create Draft Email:** Save the generated email as a draft in Gmail for human review or later sending.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Initiates the workflow manually for testing or execution.
- **Nodes Involved:**  
  - When clicking ‘Test workflow’
- **Node Details:**

| Node Name                      | Details                                                                                              |
|--------------------------------|----------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’   | Type: Manual Trigger. Role: Starts the workflow execution. Configured with default settings, no parameters. Input: None. Output: Triggers next node "Get Contacts". Failure cases: None expected. |

---

#### 1.2 Customer Retrieval

- **Overview:** Retrieves a filtered list of customers from HubSpot CRM to target for outreach.
- **Nodes Involved:**  
  - Get Contacts  
  - For Each Contact  
  - Contact Ref  
  - Sticky Note5 (comment)
- **Node Details:**

| Node Name     | Details                                                                                                                      |
|---------------|------------------------------------------------------------------------------------------------------------------------------|
| Get Contacts  | Type: HubSpot node. Role: Searches contacts with filter on "hs_buying_role" equals "DECISION_MAKER". Configured to use HubSpot App Token credentials. Input: Trigger from manual node. Output: List of filtered contacts. Failure cases: API auth errors, rate limits, empty results. |
| For Each Contact | Type: SplitInBatches. Role: Iterates over each contact individually for processing. Configured with default batch size. Input: Contacts from "Get Contacts". Output: Single contact item per iteration. Failure cases: Empty input, batch processing errors. |
| Contact Ref   | Type: NoOp. Role: Passes current contact data forward. Input: From "For Each Contact". Output: To "Variables" node. Failure cases: None. |
| Sticky Note5  | Provides context: "Get Targeted Existing Customers" with explanation about CRM filtering or CSV alternatives. |

---

#### 1.3 Loop Through Each Customer & Variable Setup

- **Overview:** Sets variables per customer to be used downstream, including first name, last name, email, and product description.
- **Nodes Involved:**  
  - Variables
- **Node Details:**

| Node Name | Details                                                                                                                         |
|-----------|---------------------------------------------------------------------------------------------------------------------------------|
| Variables | Type: Set node. Role: Defines key variables for the current customer: firstname, lastname, email, and product_to_sell description. Uses expressions to extract from current item JSON properties. Input: From "Contact Ref". Output: To "Get All Customer's Correspondence". Failure cases: Missing properties in input JSON. |

---

#### 1.4 Retrieve Customer Emails

- **Overview:** Fetches up to 20 recent emails from Gmail that were received from the current customer.
- **Nodes Involved:**  
  - Get All Customer's Correspondence  
  - Sticky Note (comment)
- **Node Details:**

| Node Name                  | Details                                                                                                                               |
|----------------------------|---------------------------------------------------------------------------------------------------------------------------------------|
| Get All Customer's Correspondence | Type: Gmail node. Role: Retrieves emails filtered by "from" address matching the customer email. Configured with OAuth2 Gmail credentials. Limit set to 20 emails per customer. Input: From "Variables". Output: Email messages array. Failure cases: Gmail API auth errors, rate limits, no emails found. |
| Sticky Note                 | Explains the value of emails as research material for customer persona building. Advises on sharing workload or CRM integration. |

---

#### 1.5 Build Customer Persona

- **Overview:** Uses AI to analyze the customer's email correspondence and extract a detailed persona with attributes relevant to sales approach.
- **Nodes Involved:**  
  - Analyse and Build Persona  
  - Google Gemini Chat Model  
  - Sticky Note1 (comment)
- **Node Details:**

| Node Name             | Details                                                                                                                                                                                                                   |
|-----------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Analyse and Build Persona | Type: Information Extractor (AI). Role: Processes concatenated email subjects, dates, and message snippets to extract persona attributes such as decision-making style, communication preferences, pain points, goals, work style, personality traits, buying behavior, business culture, and industry awareness. Uses a system prompt to instruct the AI on the task. Input: Emails from "Get All Customer's Correspondence" passed through Google Gemini Chat Model. Output: Persona attributes JSON. Failure cases: AI prompt failures, incomplete email data, API errors. |
| Google Gemini Chat Model | Type: Google Gemini Chat Model (AI). Role: Provides the AI language model interface to process email data for persona extraction. Uses Google PaLM API credentials. Input: Emails from Gmail node. Output: Processed data to "Analyse and Build Persona". Failure cases: API quota, auth issues. |
| Sticky Note1           | Describes the persona building step and the use of AI Information Extractor node to guide attribute extraction. |

---

#### 1.6 Generate Personalized Sales Email

- **Overview:** Uses the constructed persona and product details to generate a customized outreach email with subject and body.
- **Nodes Involved:**  
  - Generate Sales Email  
  - Google Gemini Chat Model1  
  - Sticky Note2 (comment)  
  - Sticky Note (general)
- **Node Details:**

| Node Name             | Details                                                                                                                                                                                                                   |
|-----------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Generate Sales Email   | Type: Information Extractor (AI). Role: Uses the persona profile and product description to generate a sales email draft. The prompt instructs the AI to mimic the customer's communication style and address their values. Outputs subject and HTML body fields. Input: Persona JSON from "Analyse and Build Persona" node. Output: Sales email content. Failure cases: AI prompt errors, incomplete persona data, API failures. |
| Google Gemini Chat Model1 | Type: Google Gemini Chat Model (AI). Role: Provides AI language model interface to generate the sales email. Uses Google PaLM API credentials. Input: Persona data. Output: Sales email JSON. Failure cases: API limits, auth issues. |
| Sticky Note2           | Details the generation of sales pitch based on persona to improve appeal and effectiveness. |

---

#### 1.7 Create Draft Email

- **Overview:** Saves the generated sales email as a draft in Gmail for human review before sending.
- **Nodes Involved:**  
  - Create Draft Email For Review  
  - Sticky Note3 (comment)
- **Node Details:**

| Node Name               | Details                                                                                                               |
|-------------------------|-----------------------------------------------------------------------------------------------------------------------|
| Create Draft Email For Review | Type: Gmail node. Role: Creates an email draft with the generated subject and HTML body. Sends to the customer's email address. Uses OAuth2 Gmail credentials. Input: Sales email JSON from "Generate Sales Email". Output: Confirmation of draft creation. Failure cases: Gmail API auth errors, invalid email addresses. |
| Sticky Note3            | Explains the draft creation step for human review to ensure quality and customization. |

---

### 3. Summary Table

| Node Name                      | Node Type                          | Functional Role                              | Input Node(s)                 | Output Node(s)                     | Sticky Note                                                                                              |
|--------------------------------|----------------------------------|----------------------------------------------|------------------------------|----------------------------------|--------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’   | Manual Trigger                   | Starts workflow execution                     | None                         | Get Contacts                     |                                                                                                        |
| Get Contacts                   | HubSpot                         | Retrieves targeted contacts                    | When clicking ‘Test workflow’ | For Each Contact                 | ## 1. Get Targeted Existing Customers<br>As with all campaigns, it's good to have a targeted subset of customers to aim for to assess the response. Here, we can pull them out of a CRM like Hubspot if granular filtering is required for example but even a simple csv of contacts would also work. |
| For Each Contact              | SplitInBatches                  | Loops through each contact                     | Get Contacts                  | Contact Ref                     |                                                                                                        |
| Contact Ref                  | NoOp                            | Passes current contact data                    | For Each Contact              | Variables                      |                                                                                                        |
| Variables                   | Set                             | Sets variables per customer (name, email, product) | Contact Ref                  | Get All Customer's Correspondence |                                                                                                        |
| Get All Customer's Correspondence | Gmail                          | Retrieves emails from Gmail for customer      | Variables                    | Google Gemini Chat Model         | ## 2. Research Customer via Emails<br>Emails can be a great source of research on how a customer or potential customer thinks, behaves and communicates. This template does require some interaction beforehand but this should could be shared amongst colleagues or a CRM. |
| Google Gemini Chat Model     | Google Gemini Chat Model (AI)   | Processes emails for persona analysis          | Get All Customer's Correspondence | Analyse and Build Persona       | ## 3. Build Persona Outline from Research<br>Once we gather all the emails, we can use AI to analyse and construct a quick persona on our customer. Personas are useful to understand the customer's position and how favourably they might respond to a product and/or service. The Information Extractor node is used to guide the LLM for attributes we're interested in. |
| Analyse and Build Persona    | Information Extractor (AI)       | Extracts persona attributes from emails       | Google Gemini Chat Model      | Generate Sales Email            |                                                                                                        |
| Generate Sales Email         | Information Extractor (AI)       | Generates personalized sales email             | Analyse and Build Persona     | Create Draft Email For Review   | ## 4. Generate Sales Pitch based on Persona<br>Using the persona, we can again ask AI to generate the perfect sales email which takes into consideration the customer's beliefs, values and communication style. In this way, each sales email can be carefully written to improve its appeal to the customer. |
| Google Gemini Chat Model1    | Google Gemini Chat Model (AI)   | AI model for sales email generation            | Analyse and Build Persona     | Generate Sales Email            |                                                                                                        |
| Create Draft Email For Review | Gmail                          | Creates draft email in Gmail for review        | Generate Sales Email          | For Each Contact               | ## 5. Create Draft for Human Review<br>Finally, an email draft is created to store the generated sales pitch for human review. If given, a list of customers to target, a SDR can ensure customised outreach in minutes rather than hours or days. |
| Sticky Note                  | Sticky Note                    | Notes on research via emails                    | None                         | None                          | ## 2. Research Customer via Emails<br>Emails can be a great source of research on how a customer or potential customer thinks, behaves and communicates. This template does require some interaction beforehand but this should could be shared amongst colleagues or a CRM. |
| Sticky Note1                 | Sticky Note                    | Notes on building persona outline               | None                         | None                          | ## 3. Build Persona Outline from Research<br>Once we gather all the emails, we can use AI to analyse and construct a quick persona on our customer. Personas are useful to understand the customer's position and how favourably they might respond to a product and/or service. The Information Extractor node is used to guide the LLM for attributes we're interested in. |
| Sticky Note2                 | Sticky Note                    | Notes on sales pitch generation                  | None                         | None                          | ## 4. Generate Sales Pitch based on Persona<br>Using the persona, we can again ask AI to generate the perfect sales email which takes into consideration the customer's beliefs, values and communication style. In this way, each sales email can be carefully written to improve its appeal to the customer. |
| Sticky Note3                 | Sticky Note                    | Notes on draft creation for review               | None                         | None                          | ## 5. Create Draft for Human Review<br>Finally, an email draft is created to store the generated sales pitch for human review. If given, a list of customers to target, a SDR can ensure customised outreach in minutes rather than hours or days. |
| Sticky Note5                 | Sticky Note                    | Notes on retrieving targeted customers           | None                         | None                          | ## 1. Get Targeted Existing Customers<br>As with all campaigns, it's good to have a targeted subset of customers to aim for to assess the response. Here, we can pull them out of a CRM like Hubspot if granular filtering is required for example but even a simple csv of contacts would also work. |
| Sticky Note4                 | Sticky Note                    | General workflow overview and usage instructions | None                         | None                          | ## Try it out<br>### This n8n template uses existing emails from customers as context to customise and "finetune" outreach emails to them using AI. [Discord](https://discord.com/invite/XPKeKXeB7d) [Forum](https://community.n8n.io/) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Node Type: Manual Trigger  
   - Name: When clicking ‘Test workflow’  
   - No parameters needed.

2. **Add HubSpot Node to Get Contacts**  
   - Node Type: HubSpot  
   - Name: Get Contacts  
   - Operation: Search contacts  
   - Authentication: App Token (configure HubSpot App Token credentials)  
   - Filter: Property "hs_buying_role" equals "DECISION_MAKER"  
   - Connect output of Manual Trigger to this node.

3. **Add SplitInBatches Node to Loop Through Contacts**  
   - Node Type: SplitInBatches  
   - Name: For Each Contact  
   - Default batch size (1) to process contacts one at a time  
   - Connect output of Get Contacts to this node.

4. **Add NoOp Node to Pass Contact Data**  
   - Node Type: NoOp  
   - Name: Contact Ref  
   - Connect output of second output (index 1) of For Each Contact to this node.

5. **Add Set Node to Define Variables**  
   - Node Type: Set  
   - Name: Variables  
   - Assign variables:  
     - firstname = `={{ $json.properties.firstname }}`  
     - lastname = `={{ $json.properties.lastname }}`  
     - email = `={{ $json.properties.email }}`  
     - product_to_sell = `"AI partnerships: a consulting package of AI development and services. We help customers find a strong foothold on AI initiatives bringing them to life cost effectively and always with results."`  
   - Connect output of Contact Ref to this node.

6. **Add Gmail Node to Retrieve Customer Emails**  
   - Node Type: Gmail  
   - Name: Get All Customer's Correspondence  
   - Operation: Get All Emails  
   - Filters: Query = `from:{{ $json.email }}` (filters emails from current customer)  
   - Limit: 20 emails  
   - Credentials: Configure Gmail OAuth2 credentials  
   - Connect output of Variables node to this node.

7. **Add Google Gemini Chat Model Node for AI Processing**  
   - Node Type: Google Gemini Chat Model (Langchain)  
   - Name: Google Gemini Chat Model  
   - Model Name: `models/gemini-2.0-flash`  
   - Credentials: Configure Google PaLM API credentials  
   - Connect output of Gmail node to this node.

8. **Add Information Extractor Node to Build Persona**  
   - Node Type: Information Extractor (Langchain)  
   - Name: Analyse and Build Persona  
   - Text Input: Concatenate email subjects, dates, and message snippets with expression:  
     ```js
     {{$input.all().map(item => `subject: ${item.json.subject}\ndate: ${$json.headers.date}\nmessage: ${item.json.text.substr(0, item.json.text.indexOf('> wrote:') ?? item.json.text.length).replace(/^On[\w\W]+$/im, '')}`).join('\n---\n')}}
     ```  
   - System Prompt Template:  
     ```
     Your task is to build a persona of a customer or potential customer so that we may better serve them for our business. Analyse the recent correspondence of the user, {{ $('Variables').item.json.email }}, and extract the required attributes.
     ```  
   - Attributes to extract (all required): decision_making_style, communication_preferences, pain_points_challenges, professional_goals_motivations, work_style_preferences, personality_behavioral_traits, buying_investment_behavior, preferred_business_culture_ethics, industry_competitive_awareness  
   - Execute Once: true  
   - Connect AI Language Model input from Google Gemini Chat Model node.  
   - Connect output to next node.

9. **Add Second Google Gemini Chat Model Node for Sales Email Generation**  
   - Node Type: Google Gemini Chat Model (Langchain)  
   - Name: Google Gemini Chat Model1  
   - Model Name: `models/gemini-2.0-flash`  
   - Credentials: Same Google PaLM API credentials  
   - Connect output of "Analyse and Build Persona" node to this node.

10. **Add Information Extractor Node to Generate Sales Email**  
    - Node Type: Information Extractor (Langchain)  
    - Name: Generate Sales Email  
    - Text Input:  
      ```
      # Profile of {{ $('Variables').first().json.firstname }} {{ $('Variables').first().json.lastname }}
      {{ Object.keys($json.output).map(key => `## ${key}\n${$json.output[key]}`).join('\n') }}
      ```  
    - System Prompt Template:  
      ```
      You are a sales representative drafting an email to close a potential customer on the following product: <product>{{ $('Variables').first().json.product_to_sell }}</product>

      Use the provided profile to draft the a suitable email which reflects similar communication style and addresses their values, ultimately convinces the customer to inquire about and/or buy this product. Provide only the subject and body of the message as this text will go into a template. Omit the subject and signature.
      ```  
    - Attributes to extract (required): subject, body (HTML formatted)  
    - Connect AI Language Model input from Google Gemini Chat Model1 node.

11. **Add Gmail Node to Create Draft Email**  
    - Node Type: Gmail  
    - Name: Create Draft Email For Review  
    - Operation: Create Draft  
    - Email Type: HTML  
    - Subject: `={{ $json.output.subject }}`  
    - Message Body: `={{ $json.output.body }}`  
    - Send To: `={{ $('Variables').first().json.email }}`  
    - Credentials: Gmail OAuth2 account  
    - Connect output of Generate Sales Email node to this node.

12. **Connect Output of Create Draft Email back to For Each Contact**  
    - Connect output of "Create Draft Email For Review" node to the input of "For Each Contact" node to continue looping.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                           | Context or Link                                                                                     |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This n8n template uses existing emails from customers as context to customise and "finetune" outreach emails to them using AI. It leverages HubSpot CRM, Gmail, and Google Gemini AI models to build customer personas and generate personalized sales emails. Adjust filters and text parsing to avoid leaking sensitive data to AI.                                                                 | Workflow description and usage instructions                                                       |
| Join the n8n community for support and discussion: [Discord](https://discord.com/invite/XPKeKXeB7d) and [Forum](https://community.n8n.io/)                                                                                                                                                                                                                                             | Community support links                                                                            |
| Consider using other CRMs or CSV files as customer sources if HubSpot is not available. Also, additional customer data like past deals can enhance AI persona building.                                                                                                                                                                                                              | Customization suggestions                                                                         |
| Emails are a rich source for understanding customer communication style, decision-making, and preferences, which can greatly improve outreach effectiveness when incorporated into AI-generated messages.                                                                                                                                                                             | Sticky Notes on research and persona building                                                    |
| The workflow supports human review by creating draft emails in Gmail before sending, allowing sales reps to customize or approve messages. Optionally, the draft step can be skipped to send emails directly.                                                                                                                                                                         | Workflow flexibility note                                                                         |

---

This documentation provides a complete, clear, and structured understanding of the workflow, enabling advanced users and AI agents to reproduce, modify, and troubleshoot it effectively.
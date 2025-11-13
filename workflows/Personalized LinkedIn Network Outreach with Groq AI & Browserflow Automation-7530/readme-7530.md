Personalized LinkedIn Network Outreach with Groq AI & Browserflow Automation

https://n8nworkflows.xyz/workflows/personalized-linkedin-network-outreach-with-groq-ai---browserflow-automation-7530


# Personalized LinkedIn Network Outreach with Groq AI & Browserflow Automation

### 1. Workflow Overview

This workflow automates personalized LinkedIn network outreach targeting automation professionals in Lagos, Nigeria. It leverages AI-powered message generation combined with LinkedIn profile scraping and automated message sending to build genuine professional relationships within the local automation community.

The workflow consists of the following logical blocks:

- **1.1 Input Reception & Profile Scraping:** Captures user input parameters via a web form and scrapes LinkedIn profiles matching the search criteria.
- **1.2 Data Processing & Batch Control:** Splits the scraped profile data into individual items and limits the batch size for manageable processing.
- **1.3 AI-Powered Message Generation:** Uses a Groq LLaMA language model and an AI Agent node to analyze each profile and generate personalized LinkedIn connection and email messages.
- **1.4 Output Formatting:** Processes the AI output JSON to extract structured message data.
- **1.5 LinkedIn Message Sending:** Automates sending LinkedIn connection requests with the personalized messages.
- **1.6 Error Handling and Observability:** Includes error detection and notes for potential workflow improvements.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Profile Scraping

**Overview:**  
This block starts the workflow from user input and scrapes LinkedIn profiles based on search terms and location.

**Nodes Involved:**  
- On form submission  
- Scrape profiles from a linkedin search

**Node Details:**

- **On form submission**  
  - *Type:* Form Trigger  
  - *Role:* Receives search parameters: Search Term, City, and Country from a user-submitted web form.  
  - *Configuration:* Form titled "Linkedin job form" with three required fields (Search Term, City, Counrty [sic]).  
  - *Input:* Web form submission (HTTP webhook).  
  - *Output:* JSON object containing user input fields for downstream use.  
  - *Potential failures:* Missing required fields, webhook misconfiguration, form submission errors.

- **Scrape profiles from a linkedin search**  
  - *Type:* Browserflow node (LinkedIn Scraper)  
  - *Role:* Scrapes up to 4 pages of LinkedIn profiles matching the search term, city, and country from the form input.  
  - *Configuration:* Dynamically references form inputs for searchTerm, city, country; nrOfPages fixed at 4.  
  - *Credentials:* Uses Browserflow API credentials.  
  - *Input:* Receives form data from previous node.  
  - *Output:* Returns an array of profile data objects.  
  - *Potential failures:* LinkedIn scraping blocked or rate limited, credential expiration, changes in LinkedIn page structure.

---

#### 2.2 Data Processing & Batch Control

**Overview:**  
Prepares scraped profile data for AI processing by splitting the array into individual items and limiting the batch size to 3 profiles at a time to avoid overload.

**Nodes Involved:**  
- Split Out1  
- Limit

**Node Details:**

- **Split Out1**  
  - *Type:* Split Out  
  - *Role:* Converts the array of profiles into individual items for sequential processing.  
  - *Configuration:* Splits out the "data" field from the scraped profiles.  
  - *Input:* Array of profiles from scraping node.  
  - *Output:* Individual profile objects passed downstream.  
  - *Failures:* If input data structure changes, the split may fail.

- **Limit**  
  - *Type:* Limit  
  - *Role:* Restricts processing to the first 3 profiles per workflow run to control resource use and API limits.  
  - *Configuration:* maxItems set to 3.  
  - *Input:* Individual profile items from Split Out1.  
  - *Output:* Limited set of profile items forwarded.  
  - *Failures:* None expected; misconfiguration could limit too few or too many items.

---

#### 2.3 AI-Powered Message Generation

**Overview:**  
Generates personalized LinkedIn and email outreach messages for each profile using a Groq LLaMA language model combined with a LangChain AI Agent configured with a detailed prompt.

**Nodes Involved:**  
- Groq Chat Model  
- AI Agent

**Node Details:**

- **Groq Chat Model**  
  - *Type:* LangChain Language Model (Groq LLaMA)  
  - *Role:* Provides advanced AI language generation capabilities as backend for the AI Agent.  
  - *Configuration:* Default options; credentials configured with Groq API keys.  
  - *Input:* AI prompt from AI Agent node.  
  - *Output:* Raw AI-generated text response.  
  - *Failures:* API auth errors, quota exceeded, timeouts, malformed prompts.

- **AI Agent**  
  - *Type:* LangChain Agent Node  
  - *Role:* Orchestrates AI prompt creation and sends it to the Groq Chat Model for generating personalized outreach messages.  
  - *Configuration:*  
    - Text prompt template embedding profile fields (name, LinkedIn URL, tagline, location, summary).  
    - Detailed instructions to create LinkedIn connection and email messages with specific style, tone, and length constraints.  
    - System message defines the agent’s role and output JSON format.  
  - *Input:* Individual profile data from Limit node.  
  - *Output:* AI-generated JSON messages as raw text in a code block.  
  - *Failures:* Expression evaluation errors in prompt, AI model response format non-compliance, runtime errors.

---

#### 2.4 Output Formatting

**Overview:**  
Parses the AI-generated raw text to extract valid JSON objects containing structured message data for downstream use.

**Nodes Involved:**  
- Code1

**Node Details:**

- **Code1**  
  - *Type:* Code (JavaScript)  
  - *Role:* Extracts JSON block wrapped in ```json ... ``` from AI Agent output and parses it into usable JSON objects.  
  - *Configuration:* Custom JS code with error handling to fallback on raw output if parsing fails.  
  - *Input:* Raw AI output text from AI Agent.  
  - *Output:* Parsed JSON objects or error objects for debugging.  
  - *Failures:* JSON parsing errors if AI output deviates from expected format, missing or malformed JSON blocks.

---

#### 2.5 LinkedIn Message Sending

**Overview:**  
Sends the personalized LinkedIn connection requests automatically using Browserflow automation.

**Nodes Involved:**  
- Send a linkedin message1

**Node Details:**

- **Send a linkedin message1**  
  - *Type:* Browserflow node  
  - *Role:* Delivers the LinkedIn message_body content to the corresponding LinkedIn profile’s URL via LinkedIn messaging interface automation.  
  - *Configuration:*  
    - Message content dynamically set from parsed AI email message_body (note: uses email message body, not LinkedIn message body, potentially an error or design choice).  
    - linkedinUrl sourced from AI Agent output’s linkedin_message.linkedin_url.  
  - *Credentials:* Browserflow API credentials must be valid.  
  - *Input:* Parsed JSON from Code1.  
  - *Output:* Message send status.  
  - *Failures:* Credential issues, LinkedIn UI changes, throttling, message content exceeding limits, mismatch between message and profile URL.

---

#### 2.6 Error Handling and Observability

**Overview:**  
A sticky note documents workflow logic, strengths, current issues, and improvement suggestions.

**Nodes Involved:**  
- Sticky Note

**Node Details:**

- **Sticky Note**  
  - *Type:* Sticky Note (documentation)  
  - *Content Summary:*  
    - Describes the workflow’s flow, key components, and value proposition.  
    - Notes strengths such as batch control, AI personalization, dual-channel outreach.  
    - Highlights issues including Browserflow node errors and suggests adding delay nodes or credential fixes.  
  - *Role:* Provides contextual understanding and maintenance notes.  
  - *Failures:* None; purely informational.

---

### 3. Summary Table

| Node Name                       | Node Type                         | Functional Role                           | Input Node(s)                | Output Node(s)               | Sticky Note                                                                                                            |
|--------------------------------|----------------------------------|-----------------------------------------|-----------------------------|-----------------------------|------------------------------------------------------------------------------------------------------------------------|
| On form submission              | Form Trigger                     | Captures user input search parameters   | (Webhook trigger)            | Scrape profiles from a linkedin search | Manual Trigger → Starts the process<br>LinkedIn Scraper → Collects automation professional profiles                    |
| Scrape profiles from a linkedin search | Browserflow (LinkedIn Scraper)  | Scrapes LinkedIn profiles based on input | On form submission           | Split Out1                  | Manual Trigger → Starts the process<br>LinkedIn Scraper → Collects automation professional profiles                    |
| Split Out1                     | Split Out                       | Splits array of profiles into individual items | Scrape profiles from a linkedin search | Limit                       | Manual Trigger → Starts the process<br>Split Out → Converts array to individual items                                  |
| Limit                         | Limit                          | Limits batch size to 3 profiles          | Split Out1                  | AI Agent                   | Manual Trigger → Starts the process<br>Limit → Controls batch size (currently 3 profiles)                              |
| Groq Chat Model                | LangChain Language Model (Groq) | Provides AI text generation backend      | AI Agent (ai_languageModel) | AI Agent                   | AI-powered personalization for each contact                                                                            |
| AI Agent                      | LangChain Agent Node            | Generates personalized LinkedIn and email messages | Limit                       | Code1                      | AI-powered personalization for each contact                                                                            |
| Code1                         | Code (JavaScript)               | Parses AI-generated JSON from raw text   | AI Agent                   | Send a linkedin message1    | Error handling in the JavaScript code                                                                                  |
| Send a linkedin message1       | Browserflow (LinkedIn Message)  | Sends LinkedIn connection requests with personalized messages | Code1                      | (End)                      | Browserflow → Sends actual LinkedIn connection requests<br>May need credential configuration or API endpoint fixes     |
| Sticky Note                   | Sticky Note                    | Documentation and workflow insights      | (None)                      | (None)                     | Manual Trigger → Starts the process<br>Key Observations and Issues to Address<br>Workflow Value                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Web Form Trigger Node ("On form submission"):**  
   - Type: Form Trigger  
   - Configure form title: "Linkedin job form"  
   - Add required fields:  
     - Search Term (text, required)  
     - City (text, required)  
     - Counrty (text, required) — note the typo, keep consistent or fix  
   - Enable webhook for form submission.

2. **Add Browserflow Node to Scrape LinkedIn Profiles ("Scrape profiles from a linkedin search"):**  
   - Type: Browserflow (LinkedIn scraper)  
   - Credentials: Set Browserflow API credentials.  
   - Parameters:  
     - searchTerm = expression referencing form input `{{$json["Search Term"]}}`  
     - city = expression `{{$json.City}}`  
     - country = expression `{{$json.Counrty}}`  
     - nrOfPages = 4  
     - operation = scrapeProfilesFromSearch  

3. **Add Split Out Node ("Split Out1"):**  
   - Type: Split Out  
   - Field to split: "data" (the array of scraped profiles)  
   - Connect output of the scraper node to this node.

4. **Add Limit Node ("Limit"):**  
   - Type: Limit  
   - Set max items to 3 to limit batch processing.  
   - Connect output of Split Out1 to Limit.

5. **Set up LangChain Groq Chat Model Node ("Groq Chat Model"):**  
   - Type: LangChain Language Model (Groq LLaMA)  
   - Credentials: Configure with Groq API keys.  
   - Leave options default unless specific tuning is needed.

6. **Add LangChain Agent Node ("AI Agent"):**  
   - Type: LangChain Agent  
   - Parameters:  
     - Text prompt as provided, embedding dynamic profile fields (`{{$json.name}}`, `{{$json.linkedin_url}}`, etc.)  
     - System message configuring the AI to generate personalized LinkedIn and email outreach messages for ObisDev in Lagos.  
   - Connect Limit node output to AI Agent input.  
   - Configure AI Agent to use "Groq Chat Model" as the linked language model.

7. **Add Code Node ("Code1") to Parse AI Output:**  
   - Type: Code (JavaScript)  
   - Paste the provided code that extracts JSON from triple-backtick code blocks in the AI response and parses it.  
   - Connect AI Agent output to this node.

8. **Add Browserflow Node to Send LinkedIn Messages ("Send a linkedin message1"):**  
   - Type: Browserflow (LinkedIn messaging automation)  
   - Credentials: Use Browserflow API credentials.  
   - Parameters:  
     - message: pull from parsed email message body (`{{$json.email.message_body}}`)  
     - linkedinUrl: pull from parsed LinkedIn message URL (`{{$json.linkedin_message.linkedin_url}}`)  
     - operation: sendMessage  
   - Connect Code1 output to this node.

9. **Add a Sticky Note for documentation:**  
   - Add a sticky note node describing the workflow steps, strengths, and current issues.

10. **Connect all nodes in the following sequence:**  
    On form submission → Scrape profiles from a linkedin search → Split Out1 → Limit → AI Agent (uses Groq Chat Model as AI backend) → Code1 → Send a linkedin message1

11. **Set credentials:**  
    - Browserflow API credentials for scraping and sending messages.  
    - Groq API credentials for the language model.

12. **Test the workflow:**  
    - Submit form with valid search parameters.  
    - Verify profiles are scraped and limited to 3.  
    - Confirm AI messages are generated and parsed correctly.  
    - Ensure LinkedIn messages are sent via Browserflow.

13. **Consider adding delay nodes between AI Agent calls if rate-limiting or API limits occur.**

---

### 5. General Notes & Resources

| Note Content                                                                                                            | Context or Link                                                                                              |
|-------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| This workflow focuses on authentic, relationship-driven outreach rather than sales, specifically targeting Lagos’s automation community. | Workflow purpose and design philosophy.                                                                     |
| The AI Agent prompt template includes detailed instructions on tone, style, and output JSON format for reliable parsing. | Critical for generating consistent AI output.                                                               |
| Current known issue: final Browserflow node shows errors likely due to credential or endpoint misconfiguration.          | Suggest verifying Browserflow credentials and API endpoints.                                               |
| Suggested improvements include adding delay nodes between AI calls and enhanced error handling for LinkedIn message sending. | To improve reliability and avoid throttling.                                                                |
| The form field "Counrty" contains a typo; consider correcting to "Country" for clarity and consistency.                  | Minor correction for user input fields.                                                                     |
| Example outreach message style uses Nigerian professional communication style emphasizing mutual value and community.    | Guidance on cultural tone for AI messaging.                                                                 |
| Workflow credits: Combines n8n automation, Groq LLaMA AI model, LangChain framework, and Browserflow LinkedIn automation. | Technical stack used.                                                                                         |

---

**Disclaimer:** The provided text is extracted exclusively from an automated workflow created with n8n, an integration and automation tool. It adheres strictly to current content policies and contains no illegal, offensive, or protected material. All manipulated data is legal and publicly accessible.
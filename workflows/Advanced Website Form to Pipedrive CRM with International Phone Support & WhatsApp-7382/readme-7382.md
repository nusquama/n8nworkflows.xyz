Advanced Website Form to Pipedrive CRM with International Phone Support & WhatsApp

https://n8nworkflows.xyz/workflows/advanced-website-form-to-pipedrive-crm-with-international-phone-support---whatsapp-7382


# Advanced Website Form to Pipedrive CRM with International Phone Support & WhatsApp

### 1. Workflow Overview

This workflow automates the process of capturing advanced website form submissions and integrating them into the Pipedrive CRM system, with special handling for international phone numbers and WhatsApp messaging. It targets sales and marketing teams seeking seamless lead capture, data refinement, and engagement from global prospects via an enhanced multi-channel approach.

The workflow is logically divided into the following blocks:

- **1.1 Form Submission Reception:** Captures raw form data from a Webflow form webhook.
- **1.2 Data Refinement & Phone Number Formatting:** Cleans and standardizes input data, especially international phone numbers with country code detection and formatting.
- **1.3 Organization Detection & Creation:** Searches CRM for existing organizations and creates new ones if none found.
- **1.4 Person Record Detection & Management:** Searches for existing persons linked to organizations; creates or updates person records accordingly.
- **1.5 Deal Creation & Notes Management:** Creates deals in CRM associated with persons, attaches notes from messages and URLs.
- **1.6 Team Notification & Lead Engagement:** Sends detailed notifications to Discord and personalized WhatsApp messages to the lead.

The workflow handles four main scenarios based on the existence of organizations and persons in the CRM:

- Scenario A: Existing Organization + Existing Person  
- Scenario B: Existing Organization + New Person  
- Scenario C: New Organization + Existing Person  
- Scenario D: New Organization + New Person  

Each scenario branches into dedicated nodes to manage the appropriate data creation and update logic.

---

### 2. Block-by-Block Analysis

#### 2.1 Form Submission Reception

- **Overview:**  
  Listens for new contact form submissions via a Webflow webhook trigger and starts the workflow with the raw form data.

- **Nodes Involved:**  
  - Form Trigger

- **Node Details:**  
  - **Form Trigger**  
    - Type: Webflow Trigger (Webhook)  
    - Configuration: Connected to a specific Webflow site with OAuth2 credentials; listens to form submissions.  
    - Input: Webhook payload with form fields such as First Name, Last Name, Company, Email, Phone, Website URL, and Message.  
    - Output: Raw form data JSON containing all submitted fields.  
    - Edge Cases: Webhook failure, OAuth token expiry, missing fields in form submission.  
    - Sticky Note: Describes form completion and automatic lead data capture.

#### 2.2 Data Refinement & Phone Number Formatting

- **Overview:**  
  Extracts and normalizes form fields into a standardized format, then applies advanced international phone number formatting and country detection logic via a custom JavaScript code node.

- **Nodes Involved:**  
  - Data refinement  
  - international dialing code

- **Node Details:**  
  - **Data refinement**  
    - Type: Set Node  
    - Configuration: Maps raw form fields to defined properties with normalized names (e.g., "Prenom", "Nom", "Entreprise").  
    - Inputs: Raw webhook JSON from Form Trigger.  
    - Outputs: Refined JSON with clean keys for subsequent processing.  
    - Edge Cases: Missing or null fields, inconsistent input naming.  
    - Sticky Note: Highlights data cleaning and preparation phase.

  - **international dialing code**  
    - Type: Code Node (JavaScript)  
    - Configuration: Contains country-specific phone number patterns and formatting logic for multiple countries (France, Belgium, Germany, US, UK, etc.).  
    - Inputs: The cleaned phone number from Data refinement node.  
    - Outputs: The phone number formatted with the correct international dialing code, plus metadata about detected country and confidence level.  
    - Edge Cases: Unrecognized or malformed phone numbers, empty inputs, fallback to default French number if detection fails.  
    - Sticky Note: Detailed explanation of phone number formatting and international support.

#### 2.3 Organization Detection & Creation

- **Overview:**  
  Searches for the submitted company name in Pipedrive CRM and branches depending on whether the organization exists or not.

- **Nodes Involved:**  
  - Search an organization  
  - If Organization exists  
  - Create Organization

- **Node Details:**  
  - **Search an organization**  
    - Type: Pipedrive Node (Search)  
    - Configuration: Searches organizations by exact match on the company name from refined data.  
    - Inputs: Refined company name.  
    - Outputs: Search results with zero or more matching organizations.  
    - Edge Cases: API failures, rate limits, multiple matches.  
    - Sticky Note: Explains organization search logic.

  - **If Organization exists**  
    - Type: If Node  
    - Configuration: Checks if the organization search returned any result (exists or not).  
    - Branches:  
      - True: Organization found → proceed with existing organization flow.  
      - False: Organization not found → trigger creation of new organization.  
    - Edge Cases: False positives if exact match fails, empty results.  
    - Sticky Note: Marks flow branching based on org existence.

  - **Create Organization**  
    - Type: Pipedrive Node (Create)  
    - Configuration: Creates new organization record with the company name from refined data.  
    - Inputs: Company name for new organization.  
    - Outputs: Created organization details including unique ID.  
    - Edge Cases: API errors, duplicate creation attempts.  
    - Sticky Note: Indicates creation of new organization in scenario C or D.

#### 2.4 Person Record Detection & Management

- **Overview:**  
  Searches for the person by full name within the context of the organization and branches into scenarios based on person existence.

- **Nodes Involved:**  
  - Search Person Scenario A & B  
  - If Person exists Scenario A & B  
  - Create a person Scenario B  
  - Update a person Scenario B  
  - Search Person Scenario C & D  
  - If Person exists Scenario C & D  
  - Create a person Scenario D  
  - Update a person Scenario C  
  - Update a person Scenario D

- **Node Details:**  
  - **Search Person Scenario A & B**  
    - Type: Pipedrive Node (Search)  
    - Configuration: Searches persons by exact full name.  
    - Inputs: Refined first and last name.  
    - Outputs: Person search results.  
    - Edge Cases: Multiple persons with same name, API errors.

  - **If Person exists Scenario A & B**  
    - Type: If Node  
    - Configuration: Checks if person search found a result.  
    - Branches:  
      - True: Existing person (Scenario A).  
      - False: New person (Scenario B).  
    - Edge Cases: False negatives, empty results.

  - **Create a person Scenario B**  
    - Type: Pipedrive Node (Create)  
    - Configuration: Creates a person linked to existing organization with email; empty phone initially.  
    - Inputs: Person name, email, org_id.  
    - Outputs: Created person details.  
    - Edge Cases: API errors, duplicate entries.

  - **Update a person Scenario B**  
    - Type: Pipedrive Node (Update)  
    - Configuration: Updates newly created person with formatted phone number.  
    - Inputs: Person ID, phone number.  
    - Outputs: Updated person record.  
    - Edge Cases: Person ID missing, API errors.

  - **Search Person Scenario C & D**  
    - Type: Pipedrive Node (Search)  
    - Configuration: Searches persons by full name for new organization scenarios.  
    - Inputs: Same as above.  
    - Outputs: Search results.  
    - Edge Cases: Same as above.

  - **If Person exists Scenario C & D**  
    - Type: If Node  
    - Configuration: Branching on person existence for new organization flow.  
    - Branches:  
      - True: Existing person (Scenario C).  
      - False: New person (Scenario D).  
    - Edge Cases: Same as above.

  - **Create a person Scenario D**  
    - Type: Pipedrive Node (Create)  
    - Configuration: Creates new person linked to newly created organization.  
    - Inputs: Person name, email, org_id.  
    - Outputs: Created person details.  
    - Edge Cases: API errors.

  - **Update a person Scenario C and D**  
    - Type: Pipedrive Node (Update)  
    - Configuration: Updates person records with formatted phone and custom properties (Lead source "Growth Ai").  
    - Inputs: Person ID, phone number, custom properties.  
    - Outputs: Updated person record.  
    - Edge Cases: API errors.

- **Sticky Notes:**  
  - Highlight the management of persons across scenarios, ensuring no duplicates, proper linking, and lead source tracking.

#### 2.5 Deal Creation & Notes Management

- **Overview:**  
  Creates deals in Pipedrive associated with the person, attaches notes from the message and website URL, and supports conditional handling if these fields are present.

- **Nodes Involved:**  
  - Create Deal Scenario A, B, C, D  
  - If message Scenario A, B, C, D  
  - Note message Scenario A, B, C, D  
  - If URL Scenario A, B, C, D  
  - Note URL Scenario A, B, C, D

- **Node Details:**  
  - **Create Deal Scenario X**  
    - Type: Pipedrive Node (Create)  
    - Configuration: Creates a deal titled "<Company Name> - Lead" linked to the person.  
    - Inputs: Deal title, person_id.  
    - Outputs: Created deal details including ID.  
    - Edge Cases: API limits, missing person_id.

  - **If message Scenario X**  
    - Type: If Node  
    - Configuration: Checks if the form message field is non-empty to decide if a note should be created.  
    - Edge Cases: Empty or whitespace-only messages.

  - **Note message Scenario X**  
    - Type: Pipedrive Node (Create Note)  
    - Configuration: Attaches the message content as a note to the deal.  
    - Inputs: Deal ID, message content.  
    - Outputs: Note creation confirmation.  
    - Edge Cases: API errors.

  - **If URL Scenario X**  
    - Type: If Node  
    - Configuration: Checks if the website URL field is non-empty for note creation.  
    - Edge Cases: Invalid or malformed URLs.

  - **Note URL Scenario X**  
    - Type: Pipedrive Node (Create Note)  
    - Configuration: Attaches the submitted website URL as a note to the deal.  
    - Inputs: Deal ID, URL content.  
    - Outputs: Note creation confirmation.  
    - Edge Cases: API errors.

- **Sticky Notes:**  
  - Emphasize comprehensive lead context preservation and CRM data enrichment.

#### 2.6 Team Notification & Lead Engagement

- **Overview:**  
  Sends formatted notifications to a Discord sales channel and personalized WhatsApp messages to the lead’s international phone number. Ensures fast and professional communication for sales follow-up.

- **Nodes Involved:**  
  - Message team Scenario A, B, C, D  
  - Send message Scenario A, B, C, D

- **Node Details:**  
  - **Message team Scenario X**  
    - Type: Discord Node (Send Message)  
    - Configuration: Sends a structured notification to a specified Discord channel with full lead details (name, company, email, phone, URL, message).  
    - Inputs: Dynamic template with lead data from Data refinement.  
    - Outputs: Message sent confirmation.  
    - Edge Cases: Discord webhook errors, rate limits.

  - **Send message Scenario X**  
    - Type: WhatsApp Node (Send Message)  
    - Configuration: Sends a personalized WhatsApp text message to the lead’s formatted phone number.  
    - Parameters include phoneNumberId credential, text body templated with lead’s first name.  
    - Inputs: Phone number from international dialing code node.  
    - Outputs: Message delivery confirmation.  
    - Edge Cases: Invalid phone number, API failures, WhatsApp account limits.

- **Sticky Notes:**  
  - Outline the benefits of immediate team alerts and personalized lead engagement via WhatsApp.

---

### 3. Summary Table

| Node Name                  | Node Type            | Functional Role                            | Input Node(s)                        | Output Node(s)                            | Sticky Note                                                                                         |
|----------------------------|----------------------|-------------------------------------------|------------------------------------|------------------------------------------|---------------------------------------------------------------------------------------------------|
| Form Trigger               | Webflow Trigger       | Entry point: captures form submission     | -                                  | Data refinement                          | # Phase 1: Form Submission Trigger                                                                |
| Data refinement            | Set                   | Normalize and map form fields              | Form Trigger                       | international dialing code               | # Phase 2: Data Processing and Phone Number Formatting                                            |
| international dialing code | Code                  | Format phone with international dialing   | Data refinement                   | Search an organization                    | # Phase 2: Data Processing and Phone Number Formatting                                            |
| Search an organization     | Pipedrive (Search)    | Look for existing organization             | international dialing code         | If Organization exists                    | # Phase 3: Organization Discovery and Management                                                  |
| If Organization exists     | If                    | Branch workflow based on organization exist | Search an organization             | Search Person Scenario A & B / Create Organization | # Phase 3: Organization Discovery and Management                                                  |
| Create Organization        | Pipedrive (Create)    | Create new organization if none found     | If Organization exists (False)     | Search Person Scenario C & D              | # Scenario C: New Organization + Existing Person                                                  |
| Search Person Scenario A & B | Pipedrive (Search)  | Search person for existing org scenarios  | If Organization exists (True)      | If Person exists Scenario A & B           | # Phase 4: Person Record Management                                                               |
| If Person exists Scenario A & B | If                | Branch person existence for existing org  | Search Person Scenario A & B        | Create Deal Scenario A / Create a person Scenario B | # Phase 4: Person Record Management                                                               |
| Create a person Scenario B | Pipedrive (Create)    | Create person linked to existing org       | If Person exists Scenario A & B (False) | Create Deal Scenario B                | # Scenario B: Existing Organization + New Person                                                  |
| Create Deal Scenario B     | Pipedrive (Create)    | Create deal linked to new person           | Create a person Scenario B          | If message Scenario B                     |                                                                                                   |
| If message Scenario B      | If                    | Check if message exists to create note     | Create Deal Scenario B              | Note message Scenario B / If URL Scenario B |                                                                                                   |
| Note message Scenario B    | Pipedrive (Create Note) | Add message note to deal                   | If message Scenario B (True)        | If URL Scenario B                         |                                                                                                   |
| If URL Scenario B          | If                    | Check if URL exists to create note         | Note message Scenario B / If message Scenario B (False) | Note URL Scenario B / Message team Scenario B |                                                                                                   |
| Note URL Scenario B        | Pipedrive (Create Note) | Add website URL note to deal               | If URL Scenario B (True)            | Message team Scenario B                   |                                                                                                   |
| Message team Scenario B    | Discord (Send Message)| Notify sales team on Discord                | Note URL Scenario B / Note message Scenario B | Send message Scenario B                |                                                                                                   |
| Send message Scenario B    | WhatsApp (Send)       | Send personalized WhatsApp message to lead | Message team Scenario B             | Update a person Scenario B                 |                                                                                                   |
| Update a person Scenario B | Pipedrive (Update)    | Update person record with phone and props | Send message Scenario B             | -                                        |                                                                                                   |
| Search Person Scenario C & D | Pipedrive (Search)  | Search person for new org scenarios         | Create Organization                 | If Person exists Scenario C & D           | # Phase 4: Person Record Management                                                               |
| If Person exists Scenario C & D | If                | Branch person existence for new org        | Search Person Scenario C & D        | Create Deal Scenario C / Create a person Scenario D | # Phase 4: Person Record Management                                                               |
| Create a person Scenario D | Pipedrive (Create)    | Create person linked to new organization    | If Person exists Scenario C & D (False) | Create Deal Scenario D                | # Scenario D: New Organization + New Person                                                      |
| Create Deal Scenario D     | Pipedrive (Create)    | Create deal linked to person                | Create a person Scenario D          | If message Scenario D                     |                                                                                                   |
| If message Scenario D      | If                    | Check if message exists to create note     | Create Deal Scenario D              | Note message Scenario D / If URL Scenario D |                                                                                                   |
| Note message Scenario D    | Pipedrive (Create Note) | Add message note to deal                   | If message Scenario D (True)        | If URL Scenario D                         |                                                                                                   |
| If URL Scenario D          | If                    | Check if URL exists to create note         | Note message Scenario D / If message Scenario D (False) | Note URL Scenario D / Message team Scenario D |                                                                                                   |
| Note URL Scenario D        | Pipedrive (Create Note) | Add website URL note to deal               | If URL Scenario D (True)            | Message team Scenario D                   |                                                                                                   |
| Message team Scenario D    | Discord (Send Message)| Notify sales team on Discord                | Note URL Scenario D / Note message Scenario D | Send message Scenario D                |                                                                                                   |
| Send message Scenario D    | WhatsApp (Send)       | Send personalized WhatsApp message to lead | Message team Scenario D             | Update a person Scenario D                 |                                                                                                   |
| Update a person Scenario D | Pipedrive (Update)    | Update person record with phone and props | Send message Scenario D             | -                                        |                                                                                                   |
| Search Person Scenario A & B | Pipedrive (Search)  | Initial person search for existing org flow | If Organization exists (True)       | If Person exists Scenario A & B           |                                                                                                   |
| If Person exists Scenario A & B | If                | Branch on person existence for existing org | Search Person Scenario A & B        | Create Deal Scenario A / Create a person Scenario B |                                                                                                   |
| Create Deal Scenario A     | Pipedrive (Create)    | Create deal linked to existing person       | If Person exists Scenario A & B (True) | If message Scenario A                   | # Scenario A: Existing Organization + Existing Person                                            |
| If message Scenario A      | If                    | Check if message exists to create note     | Create Deal Scenario A              | Note message Scenario A / If URL Scenario A |                                                                                                   |
| Note message Scenario A    | Pipedrive (Create Note) | Add message note to deal                   | If message Scenario A (True)        | If URL Scenario A                         |                                                                                                   |
| If URL Scenario A          | If                    | Check if URL exists to create note         | Note message Scenario A / If message Scenario A (False) | Note URL Scenario A / Message team Scenario A |                                                                                                   |
| Note URL Scenario A        | Pipedrive (Create Note) | Add website URL note to deal               | If URL Scenario A (True)            | Message team Scenario A                   |                                                                                                   |
| Message team Scenario A    | Discord (Send Message)| Notify sales team on Discord                | Note URL Scenario A / Note message Scenario A | Send message Scenario A                |                                                                                                   |
| Send message Scenario A    | WhatsApp (Send)       | Send personalized WhatsApp message to lead | Message team Scenario A             | Update a person Scenario A                 |                                                                                                   |
| Update a person Scenario A | Pipedrive (Update)    | Update person record with phone and props | Send message Scenario A             | -                                        |                                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Webflow Form Trigger node:**  
   - Type: Webflow Trigger  
   - Credentials: Connect your Webflow OAuth2 API credentials.  
   - Configure to listen to your specific site's form submission webhook.  
   - This node triggers the workflow on each form submission.

2. **Add a Set node named "Data refinement":**  
   - Map incoming form fields to custom properties:  
     - Prenom ← `payload.data['Prénom']`  
     - Nom ← `payload.data.Nom`  
     - Entreprise ← `payload.data.Entreprise`  
     - Mail professionnel ← `payload.data['Mail professionnel']`  
     - Téléphone ← `payload.data['Téléphone pro']`  
     - URL du site internet ← `payload.data['URL du site internet']`  
     - Message ← `payload.data.Message`

3. **Add a Code node named "international dialing code":**  
   - Paste the provided JavaScript code that detects and formats international phone numbers based on country-specific patterns.  
   - Input: Use the "Téléphone" field from Data refinement.  
   - Output: Add a new property `phone_formatted` with the formatted phone number.

4. **Add a Pipedrive node "Search an organization":**  
   - Operation: Search  
   - Resource: organization  
   - Term: Use `Entreprise` from "Data refinement"  
   - Exact match enabled.  
   - Credentials: Connect your Pipedrive API credentials.

5. **Add an If node "If Organization exists":**  
   - Condition: Check if search result has a non-empty `name` field.

6. **Branch 1 (Organization exists):**  
   - Connect "If Organization exists" True output to "Search Person Scenario A & B" (Pipedrive Search person).  
   - Search term: Full name `Prenom` + `Nom` from "Data refinement".  
   - Credentials: Pipedrive API.

7. **Add If node "If Person exists Scenario A & B":**  
   - Checks if person search result has non-empty `name`.  

8. **Branch 1a (Person exists):**  
   - Connect True output to "Create Deal Scenario A" (Pipedrive Create deal).  
   - Title: `Entreprise - Lead`  
   - Person ID: from person search result.  

9. **Branch 1b (Person does not exist):**  
   - Connect False output to "Create a person Scenario B" (Pipedrive Create person).  
   - Name, Email, Org ID (from org search).  
   - Then connect to "Create Deal Scenario B" (Pipedrive Create deal).  

10. **Add If nodes "If message Scenario A" and "If message Scenario B":**  
    - Check if message exists (non-empty) to create notes.  
    - Connect to corresponding "Note message Scenario A/B" nodes.

11. **Add If nodes "If URL Scenario A" and "If URL Scenario B":**  
    - Check if URL exists (non-empty) to create notes.  
    - Connect to "Note URL Scenario A/B" nodes.

12. **Add Discord nodes "Message team Scenario A" and "Message team Scenario B":**  
    - Send formatted message with lead details.  
    - Use Discord bot credentials and configure to sales channel.

13. **Add WhatsApp nodes "Send message Scenario A" and "Send message Scenario B":**  
    - Send personalized WhatsApp message to formatted phone number.  
    - Use WhatsApp API credentials and specify phoneNumberId.

14. **Add Pipedrive Update Person nodes "Update a person Scenario A" and "Update a person Scenario B":**  
    - Update person with formatted phone and custom properties.

15. **Branch 2 (Organization does not exist):**  
    - Connect False output of "If Organization exists" to "Create Organization" (Pipedrive Create organization).  
    - Then connect to "Search Person Scenario C & D" (search person for new org).  

16. **Add If node "If Person exists Scenario C & D":**  
    - Branch on person existence for new organization scenarios.

17. **Branch 2a (Person exists):**  
    - Connect True output to "Create Deal Scenario C" (Pipedrive Create deal).  

18. **Branch 2b (Person does not exist):**  
    - Connect False output to "Create a person Scenario D" (Pipedrive Create person).  

19. **Continue with nodes for scenarios C and D similar to A and B:**  
    - Create deals, add notes for messages and URLs, send Discord and WhatsApp notifications, update person records.

20. **Ensure all branches have appropriate connections to If nodes for messages and URLs, notes creation, notifications, and updates.**

21. **Credentials Setup:**  
    - Pipedrive API: Required for all CRM interactions.  
    - Webflow OAuth2 API: For form webhook.  
    - Discord Bot API: For sending notifications to sales channel.  
    - WhatsApp API: For sending personalized messages.

22. **Testing:**  
    - Submit test forms with various phone formats and scenarios to validate all branches and error handling.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Context or Link                                                                                                                                               |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------|
| The workflow includes comprehensive international phone number formatting supporting many countries with fallback mechanisms for unrecognized formats. This ensures WhatsApp messages reach leads correctly.                                                                                                                                                                                                                                                                                                  | Explanation in international dialing code node.                                                                                                              |
| Discord notifications are set to a specific sales channel with structured content to alert the team immediately of new leads, including all contact details for rapid response.                                                                                                                                                                                                                                                                                                                              | Discord nodes Message team Scenario A-D.                                                                                                                     |
| The workflow uses Pipedrive exact matching to prevent duplicate organizations and persons, preserving CRM data integrity. Custom properties are updated for lead source tracking "Growth Ai".                                                                                                                                                                                                                                                                                                                  | Nodes updating person records.                                                                                                                                |
| The form trigger node is based on Webflow but can be adapted to other form platforms by changing the webhook source.                                                                                                                                                                                                                                                                                                                                                                                        | Sticky note near Form Trigger.                                                                                                                                |
| This workflow ensures multi-scenario coverage for lead management, handling all combinations of existing/new organizations and persons with proper deal creation and note attachments.                                                                                                                                                                                                                                                                                                                        | General workflow design and sticky notes labeling scenarios A-D.                                                                                              |
| Lead messages and website URLs are preserved as notes linked to deals to enable full context for sales teams.                                                                                                                                                                                                                                                                                                                                                                                               | Note message and Note URL nodes.                                                                                                                              |
| All WhatsApp messages are personalized with lead first names and sent using a designated phoneNumberId credential, requiring a properly configured WhatsApp Business API account.                                                                                                                                                                                                                                                                                                                            | WhatsApp nodes Send message Scenario A-D.                                                                                                                    |
| The workflow is designed for n8n version 2.0+ with certain nodes requiring version 2.2+ for advanced If node conditions and latest API credential handling.                                                                                                                                                                                                                                                                                                                                                   | Version-specific notes in If nodes and credential usage.                                                                                                     |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data manipulated is legal and publicly available.
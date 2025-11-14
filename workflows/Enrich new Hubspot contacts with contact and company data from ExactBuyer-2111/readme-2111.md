Enrich new Hubspot contacts with contact and company data from ExactBuyer

https://n8nworkflows.xyz/workflows/enrich-new-hubspot-contacts-with-contact-and-company-data-from-exactbuyer-2111


# Enrich new Hubspot contacts with contact and company data from ExactBuyer

### 1. Workflow Overview

This n8n workflow automates the enrichment of newly created HubSpot contacts by supplementing their profiles with detailed contact and company information obtained from the ExactBuyer API. The workflow is designed to enhance the value of HubSpot contact records by updating social profiles, phone numbers, email addresses, job titles, company data, and location details.

The workflow is logically organized into the following blocks:

- **1.1 HubSpot Contact Creation Trigger:** Listens for new contact creation events in HubSpot.
- **1.2 Retrieve HubSpot Contact Details:** Fetches full contact data from HubSpot based on the trigger event.
- **1.3 Data Preparation:** Extracts and sets key identifiers such as email and user ID for subsequent enrichment.
- **1.4 Email Validation:** Checks if the contact has a valid email address to proceed.
- **1.5 Enrich Contact via ExactBuyer API:** Sends the email to ExactBuyer to retrieve enriched contact and company information.
- **1.6 Update HubSpot Contact:** Updates the original HubSpot contact with enriched data from ExactBuyer.
- **1.7 Error Handling:** Handles cases where enrichment fails or no user is found.

### 2. Block-by-Block Analysis

---

#### 2.1 HubSpot Contact Creation Trigger

- **Overview:**  
  This block initiates the workflow upon the creation of a new contact in HubSpot by listening to the HubSpot trigger event.

- **Nodes Involved:**  
  - On contact created

- **Node Details:**  
  - **Node Name:** On contact created  
  - **Type:** HubSpot Trigger  
  - **Technical Role:** Listens for new contact creation events in HubSpot via webhook.  
  - **Configuration:** Uses HubSpot Developer API OAuth2 credentials with scopes carefully set as per [n8n HubSpot Trigger docs](https://docs.n8n.io/integrations/builtin/trigger-nodes/n8n-nodes-base.hubspottrigger) to ensure correct event subscription.  
  - **Key expressions:** None (trigger node)  
  - **Input connections:** None (entry point)  
  - **Output connections:** Gets connected to "Get HubSpot contact" node.  
  - **Edge Cases / Failures:**  
    - Misconfigured OAuth2 credentials or incorrect scopes lead to webhook subscription failure.  
    - API rate limits or webhook downtime could cause missed events.  
  - **Sticky Notes:** Reminder about OAuth2 credential scopes being critical.

---

#### 2.2 Retrieve HubSpot Contact Details

- **Overview:**  
  Fetches detailed contact information from HubSpot using the contact ID provided by the trigger to enable later enrichment.

- **Nodes Involved:**  
  - Get HubSpot contact

- **Node Details:**  
  - **Node Name:** Get HubSpot contact  
  - **Type:** HubSpot node (operation: get)  
  - **Technical Role:** Retrieves full details of the contact by contact ID.  
  - **Configuration:**  
    - Uses OAuth2 credential (different from trigger credential but with similar scopes).  
    - Contact ID sourced dynamically from the trigger data (`{{$json.contactId}}`).  
  - **Key expressions:** `={{ $json.contactId }}`  
  - **Input connections:** Receives input from "On contact created" node.  
  - **Output connections:** Outputs to "Set keys" node.  
  - **Edge Cases / Failures:**  
    - Contact ID missing or invalid may cause errors.  
    - OAuth2 token expiration or scope issues may cause failures.  
  - **Sticky Notes:** Emphasizes use of a different HubSpot OAuth2 credential than the trigger one with exact scopes.

---

#### 2.3 Data Preparation

- **Overview:**  
  Extracts essential identifiers such as `user_id` and `email` from the HubSpot contact details to be used for enrichment.

- **Nodes Involved:**  
  - Set keys

- **Node Details:**  
  - **Node Name:** Set keys  
  - **Type:** Set node  
  - **Technical Role:** Assigns two new variables `user_id` and `email` from the HubSpot contact JSON data.  
  - **Configuration:**  
    - `user_id` is set to the contact’s VID (`{{$json.vid}}`).  
    - `email` is extracted from the contact properties (`{{$json.properties.email?.value}}`).  
  - **Key expressions:**  
    - `user_id = {{$json.vid}}`  
    - `email = {{$json.properties.email?.value}}`  
  - **Input connections:** Receives from "Get HubSpot contact".  
  - **Output connections:** Outputs to "if found email" node.  
  - **Edge Cases / Failures:**  
    - Null or missing email property will impact downstream enrichment step.  
  - **Sticky Notes:** None.

---

#### 2.4 Email Validation

- **Overview:**  
  Checks whether the contact has a non-empty email address to decide if enrichment should proceed.

- **Nodes Involved:**  
  - if found email

- **Node Details:**  
  - **Node Name:** if found email  
  - **Type:** If node  
  - **Technical Role:** Conditional node that tests if the `email` field is non-empty.  
  - **Configuration:**  
    - Condition: `email` field is not empty (case sensitive, strict type validation).  
  - **Key expressions:** `={{ $json.email }}`  
  - **Input connections:** Receives from "Set keys".  
  - **Output connections:**  
    - True branch: proceeds to "Enrich user from ExactBuyer".  
    - False branch: no connections (implied stop or handled elsewhere).  
  - **Edge Cases / Failures:**  
    - Missing or invalid email leads to no enrichment attempt.  
  - **Sticky Notes:** None.

---

#### 2.5 Enrich Contact via ExactBuyer API

- **Overview:**  
  Queries the ExactBuyer API to enrich contact information using the email address as an identifier.

- **Nodes Involved:**  
  - Enrich user from ExactBuyer  
  - Could not find user (error handling)

- **Node Details:**  
  - **Node Name:** Enrich user from ExactBuyer  
  - **Type:** HTTP Request  
  - **Technical Role:** Calls ExactBuyer’s enrichment endpoint with email query to retrieve additional contact and company details.  
  - **Configuration:**  
    - HTTP Method: GET (default)  
    - URL: `https://api.exactbuyer.com/v1/enrich`  
    - Query Parameters:  
      - `email` set dynamically from previous node email: `{{$json.email}}`  
      - `required` set to `"work_email,personal_email,email"` to specify desired data fields.  
    - Authentication: HTTP Header Auth using ExactBuyer API key credential.  
    - On error: configured to continue error output (does not stop workflow on failure).  
  - **Key expressions:**  
    - `email = {{$json.email}}`  
    - `required = work_email,personal_email,email`  
  - **Input connections:** Receives from "if found email" (true branch).  
  - **Output connections:**  
    - Success branch: connects to "Update contact from Hubspot".  
    - Error branch: connects to "Could not find user" (NoOp node for handling).  
  - **Edge Cases / Failures:**  
    - Invalid or missing API key causes authentication errors.  
    - Network timeouts or API downtime.  
    - Email not found on ExactBuyer returns empty or error response.  
  - **Sticky Notes:** Contains reminder to add ExactBuyer API key and use email as identifier; link to ExactBuyer API guide https://docs.exactbuyer.com/contact-enrichment/enrichment.

- **Node Name:** Could not find user  
  - **Type:** NoOp (No Operation)  
  - **Technical Role:** Placeholder node to handle cases where ExactBuyer enrichment fails or user is not found.  
  - **Configuration:** Empty (no parameters).  
  - **Input connections:** Receives error output from "Enrich user from ExactBuyer".  
  - **Output connections:** None.  
  - **Edge Cases / Failures:** None (passive node).  
  - **Sticky Notes:** None.

---

#### 2.6 Update HubSpot Contact

- **Overview:**  
  Updates the HubSpot contact record with enriched data fields retrieved from ExactBuyer.

- **Nodes Involved:**  
  - Update contact from Hubspot

- **Node Details:**  
  - **Node Name:** Update contact from Hubspot  
  - **Type:** HubSpot node (operation: update)  
  - **Technical Role:** Updates contact in HubSpot with enriched fields such as gender, education, location, job title, company info, and phone numbers.  
  - **Configuration:**  
    - Email used as the identifier for update: `={{ $('Set keys').item.json.email }}`  
    - Authentication: OAuth2 credential (same as used in "Get HubSpot contact").  
    - Additional fields mapped from ExactBuyer API response (`{{$json.result}}`):  
      - gender  
      - school (first education school name)  
      - country (location country)  
      - jobTitle  
      - lastName  
      - firstName  
      - companyName  
      - companySize  
      - phoneNumber (first phone number in E164 format)  
  - **Key expressions:**  
    - Email: `={{ $('Set keys').item.json.email }}`  
    - Field mappings: e.g., `{{$json.result.gender}}`, `{{$json.result.employment?.job?.title}}`  
  - **Input connections:** Receives from "Enrich user from ExactBuyer" (success branch).  
  - **Output connections:** None (end node).  
  - **Edge Cases / Failures:**  
    - OAuth2 token expiration or insufficient scopes may prevent update.  
    - Missing fields in ExactBuyer response may result in partial updates.  
  - **Sticky Notes:** Notes that the same credential used for getting contact is used here.

---

### 3. Summary Table

| Node Name               | Node Type          | Functional Role                              | Input Node(s)          | Output Node(s)                    | Sticky Note                                                                                                                        |
|-------------------------|--------------------|----------------------------------------------|-----------------------|----------------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| On contact created       | HubSpot Trigger    | Trigger workflow on new HubSpot contact      | None                  | Get HubSpot contact              | 1. Setup OAuth2 creds using n8n docs https://docs.n8n.io/integrations/builtin/trigger-nodes/n8n-nodes-base.hubspottrigger 2. Exact scope required |
| Get HubSpot contact      | HubSpot            | Retrieves full contact details from HubSpot  | On contact created     | Set keys                        | 1. Setup OAuth2 creds using n8n docs https://docs.n8n.io/integrations/builtin/trigger-nodes/n8n-nodes-base.hubspottrigger/ 2. Different cred than trigger, exact scopes required |
| Set keys                | Set                | Extracts user_id and email for enrichment    | Get HubSpot contact    | if found email                  |                                                                                                                                   |
| if found email           | If                 | Checks presence of email to proceed           | Set keys              | Enrich user from ExactBuyer     |                                                                                                                                   |
| Enrich user from ExactBuyer | HTTP Request       | Calls ExactBuyer API to enrich contact info  | if found email         | Update contact from Hubspot, Could not find user | 1. Add ExactBuyer API key 2. Use email as identifier 3. API guide https://docs.exactbuyer.com/contact-enrichment/enrichment     |
| Could not find user      | NoOp               | Handles failed enrichment or no user found    | Enrich user from ExactBuyer (error) | None                           |                                                                                                                                   |
| Update contact from Hubspot | HubSpot            | Updates HubSpot contact with enriched data   | Enrich user from ExactBuyer | None                           | Same credential as used in getting the contact in HubSpot                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "On contact created" node:**
   - Type: HubSpot Trigger  
   - Configure OAuth2 credentials with scopes exactly matching [n8n docs](https://docs.n8n.io/integrations/builtin/trigger-nodes/n8n-nodes-base.hubspottrigger).  
   - Set event subscription to "contact creation" event.  
   - Save and activate webhook.

2. **Create "Get HubSpot contact" node:**
   - Type: HubSpot node  
   - Operation: Get  
   - Authentication: OAuth2 (separate credential from trigger with exact scopes).  
   - Contact ID parameter: set to expression `{{$json.contactId}}` from trigger.  
   - Connect output of "On contact created" to input of this node.

3. **Create "Set keys" node:**
   - Type: Set node  
   - Assign two fields:  
     - `user_id` = `{{$json.vid}}`  
     - `email` = `{{$json.properties.email?.value}}`  
   - Connect output of "Get HubSpot contact" to input of this node.

4. **Create "if found email" node:**
   - Type: If node  
   - Condition: Check if `email` field is not empty (strict, case sensitive).  
   - Connect output of "Set keys" node to input of this node.

5. **Create "Enrich user from ExactBuyer" node:**
   - Type: HTTP Request  
   - HTTP Method: GET  
   - URL: `https://api.exactbuyer.com/v1/enrich`  
   - Query Parameters:  
     - `email` = expression `{{$json.email}}`  
     - `required` = `"work_email,personal_email,email"`  
   - Authentication: HTTP Header Auth with ExactBuyer API key credential.  
   - Set error workflow to "Continue on Error" (so workflow does not stop on HTTP errors).  
   - Connect the "True" output of "if found email" to this node.

6. **Create "Update contact from Hubspot" node:**
   - Type: HubSpot node  
   - Operation: Update  
   - Email: expression `{{$('Set keys').item.json.email}}` (reference from prior Set keys node)  
   - Authentication: OAuth2 (same credential as "Get HubSpot contact").  
   - Additional fields mapping from ExactBuyer response (`{{$json.result}}`):  
     - gender: `{{$json.result.gender}}`  
     - school: `{{$json.result.education?.[0]?.school?.name}}`  
     - country: `{{$json.result.location?.country}}`  
     - jobTitle: `{{$json.result.employment?.job?.title}}`  
     - lastName: `{{$json.result.last_name}}`  
     - firstName: `{{$json.result.first_name}}`  
     - companyName: `{{$json.result.employment?.name}}`  
     - companySize: `{{$json.result.employment.size}}`  
     - phoneNumber: `{{$json.result.phone_numbers?.[0]?.E164}}`  
   - Connect success output of "Enrich user from ExactBuyer" to this node.

7. **Create "Could not find user" node:**
   - Type: NoOp node (empty)  
   - Connect error output of "Enrich user from ExactBuyer" to this node.

8. **Activate the workflow.**

9. **Credential setups:**
   - HubSpot Trigger OAuth2 credential: scopes per HubSpot trigger requirements.  
   - HubSpot OAuth2 credential for get/update: similar scopes but separate credential.  
   - ExactBuyer API key credential: HTTP Header Auth with provided API key.

---

### 5. General Notes & Resources

| Note Content                                                                                                                           | Context or Link                                                                                         |
|----------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Be careful to configure OAuth2 credentials for HubSpot with exact scopes as per n8n documentation to avoid permission issues.           | https://docs.n8n.io/integrations/builtin/credentials/hubspot/                                         |
| ExactBuyer API documentation provides detailed info on enrichment parameters and response structure.                                   | https://docs.exactbuyer.com/contact-enrichment/enrichment                                              |
| This workflow can be customized by adding additional fields returned by ExactBuyer to the HubSpot update node to enrich profiles more. |                                                                                                       |
| Workflow designed to gracefully handle enrichment failures by continuing on error and isolating failures without stopping the workflow. |                                                                                                       |
| HubSpot trigger node webhook must be activated in n8n for receiving real-time contact creation events.                                  |                                                                                                       |

---

This document provides a detailed understanding and stepwise rebuilding guidance for the ExactBuyer data enrichment workflow for HubSpot contacts in n8n. It enables users and AI agents to confidently operate, modify, and troubleshoot the workflow.
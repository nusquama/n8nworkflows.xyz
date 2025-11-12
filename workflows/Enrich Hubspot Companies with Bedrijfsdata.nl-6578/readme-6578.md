Enrich Hubspot Companies with Bedrijfsdata.nl

https://n8nworkflows.xyz/workflows/enrich-hubspot-companies-with-bedrijfsdata-nl-6578


# Enrich Hubspot Companies with Bedrijfsdata.nl

### 1. Workflow Overview

This workflow "Enrich Hubspot Companies with Bedrijfsdata.nl" is designed to automatically enrich company records in Hubspot CRM using the Bedrijfsdata.nl API. It targets companies in Hubspot that have changes in their properties (typically the domain field), retrieves additional company data from Bedrijfsdata.nl, and updates the Hubspot record accordingly.

The workflow is logically divided into these main blocks:

- **1.1 Input Reception & Validation:** Receives incoming webhook POST requests from Hubspot on company property changes. Validates the event type and filters out invalid or test events.

- **1.2 Hubspot Company Data Retrieval:** Based on the incoming Hubspot company ID, it fetches detailed company data from Hubspot. Also includes a test mode to use a fixed test company ID.

- **1.3 Decision Logic for Data Enrichment:** Determines whether to enrich company data by Bedrijfsdata.nl using an existing Bedrijfsdata ID or by searching/enriching based on company details.

- **1.4 Bedrijfsdata.nl Enrichment:** Calls Bedrijfsdata.nl API to either get company data by ID or enrich data based on domain, city, and name.

- **1.5 Data Matching and Selection:** Filters and selects the best matching company profile from the Bedrijfsdata.nl response.

- **1.6 Hubspot Company Update:** Updates the Hubspot company record with enriched data, including custom properties such as prospectpro ID, KvK (Dutch Chamber of Commerce number), LinkedIn link, etc.

- **1.7 Error Handling:** Contains nodes to catch and handle different error scenarios, such as invalid Hubspot requests, Hubspot API errors, Bedrijfsdata.nl API errors, or no matching company found.

- **1.8 Notes & Documentation:** Contains sticky notes with instructions, setup guidance, and helpful links.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Validation

**Overview:**  
Receives webhook events from Hubspot for company property changes and validates the incoming data to ensure it is a relevant event, includes a valid company ID, and comes from the expected Hubspot portal.

**Nodes Involved:**  
- Hubspot - Company - Enrichment request (Webhook)  
- Validate Incoming Data (If)  
- Is TEST Mode? (If)  
- Get TEST Company Data (Hubspot)  
- Get Company Data (Hubspot)

**Node Details:**  

- **Hubspot - Company - Enrichment request**  
  - Type: Webhook Node  
  - Role: Receives POST requests from Hubspot webhook on company property changes (e.g., domain updates).  
  - Key Config: POST method, specific webhook path.  
  - Input: HTTP request payload from Hubspot webhook.  
  - Output: JSON with event data (e.g., objectId, portalId, subscriptionType).  
  - Edge Cases: Unauthorized or malformed requests; ensure webhook URL is secured and only accepts valid Hubspot events.

- **Validate Incoming Data**  
  - Type: If Node  
  - Role: Validates incoming webhook payload with conditions:  
    - `objectId` exists and is truthy  
    - `portalId` equals 139601726 (security measure)  
    - `subscriptionType` equals "company.propertyChange"  
  - Inputs: Output from webhook.  
  - Outputs: Passes valid events forward; invalid events routed to error handling.  
  - Edge Cases: Missing or invalid objectId, wrong portal, unrelated event types.

- **Is TEST Mode?**  
  - Type: If Node  
  - Role: Detects if the incoming company ID is a test ID (specifically 123).  
  - Inputs: Validated webhook data.  
  - Outputs:  
    - True path: Use fixed test company ID data via "Get TEST Company Data" node.  
    - False path: Proceed to fetch actual company data.  
  - Edge Cases: Test mode prevents API errors from non-existent company IDs during tests.

- **Get TEST Company Data**  
  - Type: Hubspot Node (Get Company)  
  - Role: Retrieves a fixed test company record by ID (7611838964) for testing enrichment logic.  
  - Inputs: Triggered only in test mode.  
  - Outputs: Company data for enrichment.  
  - Error Handling: Continues on errors to prevent stopping workflow.

- **Get Company Data**  
  - Type: Hubspot Node (Get Company)  
  - Role: Retrieves company data from Hubspot based on the actual company ID from webhook event.  
  - Inputs: Company ID from webhook.  
  - Outputs: Company data JSON.  
  - Error Handling: Continues on errors for robustness.

---

#### 2.2 Decision Logic for Data Enrichment

**Overview:**  
Determines whether the Hubspot company record contains a known Bedrijfsdata.nl company ID (`prospectpro_id`). If yes, it uses this ID for a direct Bedrijfsdata.nl lookup. Otherwise, it performs a data enrichment search based on available company data.

**Nodes Involved:**  
- Has Known Bedrijfsdata ID (If)  
- Get company (Bedrijfsdata.nl API)  
- Enrich company data (Bedrijfsdata.nl API)

**Node Details:**  

- **Has Known Bedrijfsdata ID**  
  - Type: If Node  
  - Role: Checks if the Hubspot company contains a non-empty `prospectpro_id` property.  
  - Input: Hubspot company data.  
  - Outputs:  
    - True: Triggers "Get company" node (direct lookup).  
    - False: Triggers "Enrich company data" node (search/enrich).  
  - Edge Cases: Missing or empty `prospectpro_id` leads to enrichment path.

- **Get company**  
  - Type: Bedrijfsdata.nl Node (Get Company)  
  - Role: Retrieves company information from Bedrijfsdata.nl using the known company ID.  
  - Input: `prospectpro_id` from Hubspot company.  
  - Parameters: `details` set to true to get detailed info.  
  - Credentials: Bedrijfsdata.nl API key required.  
  - Outputs: Company profile JSON or error.  
  - Error Handling: Continues on error to allow fallback or error processing.

- **Enrich company data**  
  - Type: Bedrijfsdata.nl Node (Enrich)  
  - Role: Enriches company data by querying Bedrijfsdata.nl with domain, city, and name from Hubspot.  
  - Parameters:  
    - `url` from company domain property  
    - `city` from company city property  
    - `name` from company name property  
    - `details` true for full profile  
  - Credentials: Bedrijfsdata.nl API key required.  
  - Outputs: Enrichment results, including match counts and profiles.  
  - Error Handling: Continues on errors such as insufficient credits.

---

#### 2.3 Data Matching and Selection

**Overview:**  
Processes the results from Bedrijfsdata.nl API calls to determine if any company profile was found. If found, selects the best match for updating Hubspot.

**Nodes Involved:**  
- If the company profile is found.. (If)  
- If your data can be matched.. (If)  
- Output only the best match (Code)  
- Reformat for processing (Code)

**Node Details:**  

- **If the company profile is found..**  
  - Type: If Node  
  - Role: Checks if total companies found > 0 from "Get company" node output.  
  - Inputs: Bedrijfsdata.nl "Get company" response.  
  - Outputs:  
    - True: Proceed to select best match.  
    - False: Route to error "No company found".  
  - Edge Cases: No matches found triggers fallback or error.

- **If your data can be matched..**  
  - Type: If Node  
  - Role: Checks if number of matched companies (`found`) > 0 from enrichment results.  
  - Inputs: Bedrijfsdata.nl "Enrich company data" response.  
  - Outputs:  
    - True: Proceed to reformat data for Hubspot update.  
    - False: Route to error "No company found".  
  - Edge Cases: No matches leads to error handling.

- **Output only the best match**  
  - Type: Code Node (JavaScript)  
  - Role: Extracts the first company from the list of matched companies for update.  
  - Input: JSON with companies array.  
  - Output: JSON of the first matched company profile.  
  - Edge Cases: Empty companies array may cause undefined; should be guarded by previous If node.

- **Reformat for processing**  
  - Type: Code Node (JavaScript)  
  - Role: Extracts the company object from the response and prepares it for Hubspot update.  
  - Input: JSON with company object.  
  - Output: JSON formatted for update node.  
  - Edge Cases: Missing company property could cause errors.

---

#### 2.4 Hubspot Company Update

**Overview:**  
Updates the Hubspot company record with the enriched data from Bedrijfsdata.nl, mapping key fields like `prospectpro_id`, `kvk` (KvK number), and `linkedin_link`.

**Nodes Involved:**  
- Update a company (Hubspot)

**Node Details:**  

- **Update a company**  
  - Type: Hubspot Node (Update Company)  
  - Role: Updates Hubspot company properties with enriched data.  
  - Inputs: Reformatted Bedrijfsdata.nl company JSON and Hubspot company ID.  
  - Parameters:  
    - Company ID dynamically chosen: uses webhook objectId unless it is test ID 123, then uses test company ID.  
    - Updates custom properties:  
      - `prospectpro_id` with Bedrijfsdata.nl `id`  
      - `kvk` with Bedrijfsdata.nl `coc` (KvK number)  
      - `linkedin_link` with first LinkedIn link if array, else single string  
  - Authentication: OAuth2 with Hubspot private app credentials.  
  - Error Handling: Continues on errors.

---

#### 2.5 Error Handling

**Overview:**  
Defines no-op nodes for different error types. These nodes act as placeholders where users can implement logging, notifications, or other error management workflows.

**Nodes Involved:**  
- Error type 1: Invalid request from Hubspot  
- Error type 2 - Hubspot error  
- Error type 3: Bedrijfsdata.nl API (like insufficient credits)  
- Error type 4 - No company found

**Node Details:**  

- **Error nodes (NoOp nodes)**  
  - Type: No Operation Node (placeholder)  
  - Role: Catch and route different types of errors to allow specific handling.  
  - Context:  
    - Type 1: Invalid or unauthorized Hubspot webhook requests.  
    - Type 2: Hubspot API errors during company data retrieval or update.  
    - Type 3: Bedrijfsdata.nl API errors such as insufficient credits or request failures.  
    - Type 4: No matching company found in Bedrijfsdata.nl enrichment.  
  - Recommendation: Connect these to logging, messaging, or alerting integrations.

---

#### 2.6 Notes & Documentation

**Overview:**  
Sticky notes throughout the workflow provide detailed instructions, setup guides, and contextual information about Hubspot private app setup, webhook configuration, API usage, and best practices.

**Nodes Involved:**  
- Sticky Note (General Hubspot webhook explanation)  
- Sticky Note1 (Event filtering & security)  
- Sticky Note2 (Testing webhook in Hubspot)  
- Sticky Note3 (Sub-flow selection rationale)  
- Sticky Note4 (Bedrijfsdata.nl ID usage)  
- Sticky Note5 (Enrichment by available data)  
- Sticky Note6 (Hubspot update mapping instructions)  
- Sticky Note7 (Error handling recommendations)  
- Sticky Note8 (Post-enrichment options)  
- Sticky Note9 (Template explanation and screenshots)  
- Sticky Note10 (Hubspot private app configuration screenshots and scopes)

**Key Links:**  
- [Hubspot private apps](https://developers.hubspot.com/docs/guides/apps/private-apps/overview)  
- [Hubspot webhook subscriptions](https://developers.hubspot.com/docs/guides/apps/private-apps/create-and-edit-webhook-subscriptions-in-private-apps)  
- [Bedrijfsdata.nl Developers Platform](https://developers.bedrijfsdata.nl)  
- [Bedrijfsdata.nl API documentation](https://www.bedrijfsdata.nl)  
- ProspectPro software: https://www.prospectpro.nl

---

### 3. Summary Table

| Node Name                          | Node Type                  | Functional Role                             | Input Node(s)                         | Output Node(s)                                             | Sticky Note                                                                                                                                |
|-----------------------------------|----------------------------|--------------------------------------------|-------------------------------------|------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| Hubspot - Company - Enrichment request | Webhook                   | Receives Hubspot webhook events on company property changes | None                                | Validate Incoming Data                                      | ## Hubspot Trigger: Use Webhook Trigger with Hubspot private app; subscribe to company.propertyChange events. See setup screenshots.         |
| Validate Incoming Data             | If                         | Validates event type, portal ID, and objectId presence | Hubspot - Company - Enrichment request | Is TEST Mode? / Error type 1: Invalid request from Hubspot   | ## Event filtering & security: Filter appropriate events, verify Hubspot portal ID for security.                                            |
| Is TEST Mode?                     | If                         | Detects if running in test mode (company ID=123) | Validate Incoming Data              | Get TEST Company Data / Get Company Data                    | ## Testing your webhook in Hubspot: Test mode uses fixed test company ID to avoid errors caused by non-existent IDs during testing.           |
| Get TEST Company Data             | Hubspot                    | Fetches test company data by fixed ID for testing | Is TEST Mode? (true branch)         | Has Known Bedrijfsdata ID                                   |                                                                                                                                             |
| Get Company Data                  | Hubspot                    | Fetches company data from Hubspot by actual ID | Is TEST Mode? (false branch)        | Has Known Bedrijfsdata ID                                   |                                                                                                                                             |
| Has Known Bedrijfsdata ID         | If                         | Checks if company has Bedrijfsdata.nl ID    | Get TEST Company Data / Get Company Data | Get company / Enrich company data                            | ## Select the appropriate sub-flow: Use ID if available, else enrich by data.                                                                |
| Get company                      | Bedrijfsdata.nl API         | Retrieves company by Bedrijfsdata.nl ID     | Has Known Bedrijfsdata ID (true)    | If the company profile is found.. / Error type 3: Bedrijfsdata.nl API (like insufficient credits) | ## Retrieve company data by Bedrijfsdata.nl ID: IDs are persistent and recommended for updates.                                                |
| Enrich company data              | Bedrijfsdata.nl API         | Enriches company data by domain, city, name | Has Known Bedrijfsdata ID (false)   | If your data can be matched.. / Error type 3: Bedrijfsdata.nl API (like insufficient credits) | ## Retrieve company data by whatever info you have: Advanced matching algorithms used to find company profile from partial data.              |
| If the company profile is found.. | If                         | Checks if Bedrijfsdata.nl returned any results | Get company                       | Output only the best match / Error type 4 - No company found |                                                                                                                                             |
| If your data can be matched..     | If                         | Checks if enrichment found matches           | Enrich company data                | Reformat for processing / Error type 4 - No company found   |                                                                                                                                             |
| Output only the best match        | Code                       | Selects the first matched company profile    | If the company profile is found..  | Update a company                                           |                                                                                                                                             |
| Reformat for processing           | Code                       | Extracts company object for Hubspot update   | If your data can be matched..       | Update a company                                           |                                                                                                                                             |
| Update a company                  | Hubspot                    | Updates Hubspot company record with enriched data | Reformat for processing / Output only the best match | Do nothing / Error type 2 - Hubspot error                      | ## Update in Hubspot: Map Bedrijfsdata.nl fields to Hubspot custom properties. See docs for available data points.                           |
| Do nothing                      | NoOp                       | Placeholder after successful update          | Update a company                   | None                                                      | ## Enrichment completed! Optionally trigger logs or further workflows (e.g., decision makers, scraping, outreach).                           |
| Error type 1: Invalid request from Hubspot | NoOp                       | Handles invalid or unauthorized webhook requests | Validate Incoming Data (fail branch) | None                                                      | ## Error handling: Implement logging or notifications for invalid requests.                                                                   |
| Error type 2 - Hubspot error      | NoOp                       | Handles Hubspot API errors                     | Get Company Data / Update a company (error branches) | None                                                      | ## Error handling: Log Hubspot API errors or notify for troubleshooting.                                                                      |
| Error type 3: Bedrijfsdata.nl API (like insufficient credits) | NoOp                       | Handles Bedrijfsdata.nl API errors              | Get company / Enrich company data (error branches) | None                                                      | ## Error handling: Monitor Bedrijfsdata API errors such as insufficient credits.                                                              |
| Error type 4 - No company found   | NoOp                       | Handles no matching company found                | If the company profile is found.. / If your data can be matched.. (fail branches) | None                                                      | ## Error handling: Log or notify when no Bedrijfsdata.nl company match is found.                                                              |
| Sticky Note                      | StickyNote                 | Multiple notes providing setup instructions and context | None                              | None                                                      | See detailed notes below.                                                                                                                   |
| Sticky Note1                     | StickyNote                 | Event filtering & security guidance             | None                              | None                                                      |                                                                                                                                             |
| Sticky Note2                     | StickyNote                 | Testing webhook advice                            | None                              | None                                                      |                                                                                                                                             |
| Sticky Note3                     | StickyNote                 | Explanation on sub-flow selection                 | None                              | None                                                      |                                                                                                                                             |
| Sticky Note4                     | StickyNote                 | Usage of Bedrijfsdata.nl IDs                        | None                              | None                                                      |                                                                                                                                             |
| Sticky Note5                     | StickyNote                 | Enrichment based on available Hubspot data          | None                              | None                                                      |                                                                                                                                             |
| Sticky Note6                     | StickyNote                 | Hubspot update mapping details                      | None                              | None                                                      |                                                                                                                                             |
| Sticky Note7                     | StickyNote                 | Error handling recommendation                        | None                              | None                                                      |                                                                                                                                             |
| Sticky Note8                     | StickyNote                 | Post enrichment options                              | None                              | None                                                      |                                                                                                                                             |
| Sticky Note9                     | StickyNote                 | Workflow template info, screenshots                   | None                              | None                                                      |                                                                                                                                             |
| Sticky Note10                    | StickyNote                 | Hubspot private app setup screenshots & scopes        | None                              | None                                                      |                                                                                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node:**  
   - Type: Webhook  
   - Name: "Hubspot - Company - Enrichment request"  
   - Set HTTP Method: POST  
   - Set Webhook Path (e.g., random UUID or custom path)  
   - This node will receive Hubspot webhook events on company property changes.

2. **Create If Node for Validation:**  
   - Name: "Validate Incoming Data"  
   - Conditions (AND):  
     - `!!$json.body[0].objectId` is true (boolean check)  
     - `$json.body[0].portalId` equals 139601726 (your Hubspot portal ID)  
     - `$json.body[0].subscriptionType` equals "company.propertyChange"  
   - Connect webhook node output to this node.

3. **Create If Node for Test Mode:**  
   - Name: "Is TEST Mode?"  
   - Condition:  
     - `$json.body[0].objectId` equals 123 (test company ID)  
   - Connect "Validate Incoming Data" true output to this.

4. **Create Hubspot Node to Get TEST Company Data:**  
   - Type: Hubspot  
   - Operation: Get Company  
   - Company ID: 7611838964 (fixed test company)  
   - Auth: Hubspot OAuth2 credentials  
   - Connect "Is TEST Mode?" true output to this node.

5. **Create Hubspot Node to Get Actual Company Data:**  
   - Type: Hubspot  
   - Operation: Get Company  
   - Company ID: `={{ $json.body[0].objectId }}` (dynamic from webhook)  
   - Auth: Hubspot OAuth2 credentials  
   - Connect "Is TEST Mode?" false output to this node.

6. **Create If Node to Check Bedrijfsdata ID:**  
   - Name: "Has Known Bedrijfsdata ID"  
   - Condition:  
     - Check if `$json.properties.prospectpro_id.value` exists and is not empty string.  
   - Connect both "Get TEST Company Data" and "Get Company Data" nodes to this node.

7. **Create Bedrijfsdata.nl Node to Get Company by ID:**  
   - Type: Bedrijfsdata.nl API Node (Get Company)  
   - Parameters:  
     - `id`: `={{ $json.properties.prospectpro_id.value }}`  
     - `details`: true  
   - Credentials: Bedrijfsdata.nl API credentials  
   - Connect "Has Known Bedrijfsdata ID" true output to this node.

8. **Create Bedrijfsdata.nl Node to Enrich Company Data:**  
   - Type: Bedrijfsdata.nl API Node (Enrich)  
   - Parameters:  
     - `url`: `={{ $json.properties.domain.value }}`  
     - `city`: `={{ $json.properties.city.value }}`  
     - `name`: `={{ $json.properties.name.value }}`  
     - `details`: true  
   - Credentials: Bedrijfsdata.nl API credentials  
   - Connect "Has Known Bedrijfsdata ID" false output to this node.

9. **Create If Node to Check if Company Profile Found (Get Company path):**  
   - Condition: `$json.total` > 0  
   - Connect "Get company" node output to this.

10. **Create If Node to Check if Data Matched (Enrich path):**  
    - Condition: `$json.found` > 0  
    - Connect "Enrich company data" node output to this.

11. **Create Code Node "Output only the best match":**  
    - Extract the first company from `companies` array.  
    - JS code example:  
      ```js
      const input = items[0].json;
      const firstCompany = input.companies?.[0];
      return [{ json: firstCompany }];
      ```  
    - Connect "If the company profile is found.." true output to this node.

12. **Create Code Node "Reformat for processing":**  
    - Extract company object from response:  
      ```js
      const input = items[0].json;
      const company = input.company;
      return [{ json: company }];
      ```  
    - Connect "If your data can be matched.." true output to this node.

13. **Create Hubspot Node to Update Company:**  
    - Operation: Update Company  
    - Company ID Expression:  
      ```js
      ={{ $('Hubspot - Company - Enrichment request').first().json.body[0].objectId !== 123 ? $('Hubspot - Company - Enrichment request').first().json.body[0].objectId : $('Get TEST Company Data').first().json.companyId }}
      ```  
    - Update Fields (Custom Properties):  
      - `prospectpro_id` = `={{ $json.id }}`  
      - `kvk` = `={{ $json.coc }}`  
      - `linkedin_link` = `={{ Array.isArray($json.linkedin_link) ? $json.linkedin_link[0] : $json.linkedin_link }}`  
    - Auth: Hubspot OAuth2 credentials  
    - Connect "Output only the best match" and "Reformat for processing" nodes to this update node.

14. **Create NoOp Nodes for Error Handling:**  
    - Create four NoOp nodes named accordingly:  
      - "Error type 1: Invalid request from Hubspot"  
      - "Error type 2 - Hubspot error"  
      - "Error type 3: Bedrijfsdata.nl API (like insufficient credits)"  
      - "Error type 4 - No company found"  
    - Connect fail branches of checks and API calls to appropriate error nodes.

15. **Create "Do nothing" NoOp Node:**  
    - Connect successful update node's success output to this node as end point.

16. **(Optional) Add Sticky Notes:**  
    - Add sticky notes with instructions, links, and screenshots as per documentation for clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                      | Context or Link                                                                                      |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Hubspot private app setup is recommended over public apps for easier webhook subscription and authentication. The webhook must subscribe to **company.propertyChange** events to trigger enrichment.                                                                                                                                                                               | https://developers.hubspot.com/docs/guides/apps/private-apps/overview                              |
| When testing the webhook, Hubspot may send non-existing company IDs; use a fixed test company ID to avoid API errors in such cases.                                                                                                                                                                                                                                            | Hubspot webhook testing best practice                                                              |
| Bedrijfsdata.nl API allows two approaches: fetching by persistent company ID (`prospectpro_id`), or enriching by partial data (domain, city, name). Using ID is preferred for accuracy and updates.                                                                                                                                                                                 | https://developers.bedrijfsdata.nl                                                                 |
| The workflow includes detailed error handling placeholders; implement logging or notifications (e.g., Slack, Email) to monitor failures effectively.                                                                                                                                                                                                                             | Error handling best practices                                                                       |
| Bedrijfsdata.nl offers extensive company data points; map needed properties to Hubspot custom fields as per your CRM schema.                                                                                                                                                                                                                                                    | https://docs.bedrijfsdata.nl                                                                        |
| After enrichment, you may trigger further workflows such as searching decision makers, scraping websites, or automated outreach.                                                                                                                                                                                                                                                | Workflow extension ideas                                                                            |
| ProspectPro is Bedrijfsdata.nl’s B2B prospecting software that provides company IDs and datasets which can be integrated with this workflow.                                                                                                                                                                                                                                     | https://www.prospectpro.nl                                                                          |

---

**Disclaimer:**  
The provided text and workflow exclusively originate from an automated n8n workflow. It strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.
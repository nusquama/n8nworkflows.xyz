Sync Dartagnan Email Templates to Braze

https://n8nworkflows.xyz/workflows/sync-dartagnan-email-templates-to-braze-3081


# Sync Dartagnan Email Templates to Braze

### 1. Workflow Overview

This workflow automates the synchronization of email templates from Dartagnan to Braze, ensuring consistent email marketing content across both platforms. It is designed for marketing teams and agencies who want to maintain brand consistency, reduce manual effort, and avoid errors in template transfers.

The workflow is logically divided into the following blocks:

- **1.1 Trigger and Credentials Initialization**: Scheduled trigger and credential assignment to start the workflow and prepare API access.
- **1.2 Authentication**: Obtaining an OAuth2 access token from Dartagnan for API calls.
- **1.3 Template Discovery**: Fetching existing email templates from Braze and Dartagnan projects and campaigns.
- **1.4 Template Comparison and Filtering**: Comparing Dartagnan templates with Braze templates to identify which templates need updating or creation.
- **1.5 Template Content Processing**: Retrieving HTML and media content from Dartagnan, embedding direct image URLs, and encoding content.
- **1.6 Template Synchronization**: Creating new templates or updating existing templates in Braze via API calls.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and Credentials Initialization

**Overview:**  
This block triggers the workflow on a schedule and assigns the necessary API credentials for Dartagnan and Braze.

**Nodes Involved:**  
- Every 5 minutes start  
- Assign Credentials  
- Sticky Note (Authentication Set Up)  
- Sticky Note (Trigger)  

**Node Details:**  

- **Every 5 minutes start**  
  - Type: Schedule Trigger  
  - Role: Starts the workflow every 5 minutes (configurable interval)  
  - Configuration: Interval set to 5 minutes  
  - Inputs: None (trigger node)  
  - Outputs: Connects to Assign Credentials  
  - Edge Cases: If n8n is down or paused, scheduled triggers won't fire; rate limits must be respected.

- **Assign Credentials**  
  - Type: Set  
  - Role: Stores Dartagnan client_id, client_secret, Braze instance_url, and API key as static values (to be replaced by user)  
  - Configuration: Four string parameters for credentials, placeholders prompting user input  
  - Inputs: From trigger  
  - Outputs: Connects to Token Request and List Available Email Template Braze  
  - Edge Cases: Missing or incorrect credentials will cause authentication failures downstream.

- **Sticky Notes**  
  - Provide contextual information about authentication and trigger setup.

---

#### 2.2 Authentication

**Overview:**  
Obtains an OAuth2 access token from Dartagnan using client credentials for subsequent API calls.

**Nodes Involved:**  
- Token Request  
- Sticky Note (Authentication Token)  

**Node Details:**  

- **Token Request**  
  - Type: HTTP Request  
  - Role: Requests OAuth2 token from Dartagnan's token endpoint  
  - Configuration: POST request to `https://app.dartagnan.io/oauth/v2/token` with body parameters: client_id, client_secret, grant_type=client_credentials  
  - Expressions: Uses credentials from "Assign Credentials" node for client_id and client_secret  
  - Inputs: From Assign Credentials  
  - Outputs: Connects to Dartagnan Project list  
  - Edge Cases: Token request failure due to invalid credentials, network issues, or rate limiting.

---

#### 2.3 Template Discovery

**Overview:**  
Retrieves the list of existing email templates from Braze and Dartagnan projects and campaigns to prepare for comparison.

**Nodes Involved:**  
- List Available Email Template Braze  
- Split Out  
- Filtering Braze Email Template  
- Dartagnan Project list  
- Filtered Project Campaign  
- Filtering Dartagnan Campaigns  
- Sticky Notes (Template Discovery)  

**Node Details:**  

- **List Available Email Template Braze**  
  - Type: HTTP Request  
  - Role: Fetches all email templates from Braze via their API  
  - Configuration: GET request to `https://{{instance_url}}/templates/email/list` with Authorization header using Braze API key  
  - Inputs: From Assign Credentials  
  - Outputs: Connects to Split Out  
  - Edge Cases: API errors, rate limits, invalid API key.

- **Split Out**  
  - Type: Split Out  
  - Role: Splits the array of templates from Braze into individual items for processing  
  - Configuration: Splits on the "templates" field  
  - Inputs: From List Available Email Template Braze  
  - Outputs: Connects to Filtering Braze Email Template  
  - Edge Cases: Empty or malformed response arrays.

- **Filtering Braze Email Template**  
  - Type: Set  
  - Role: Maps Braze template fields to simplified variables for comparison  
  - Configuration: Assigns braze_template_name, email_template_id, created_at, updated_at from input JSON  
  - Inputs: From Split Out  
  - Outputs: Connects to Existing In Braze node (merge)  
  - Edge Cases: Missing fields or unexpected data formats.

- **Dartagnan Project list**  
  - Type: HTTP Request  
  - Role: Retrieves list of Dartagnan projects with Authorization header using Dartagnan token  
  - Configuration: GET request to `https://app.dartagnan.io/api/public/projects`  
  - Inputs: From Token Request  
  - Outputs: Connects to Filtered Project Campaign  
  - Edge Cases: Token expiration, network errors.

- **Filtered Project Campaign**  
  - Type: HTTP Request  
  - Role: Retrieves detailed project data including campaigns for a specific project ID  
  - Configuration: GET request to `https://app.dartagnan.io/api/public/projects/{{ $json.id }}`  
  - Inputs: From Dartagnan Project list  
  - Outputs: Connects to Filtering Dartagnan Campaigns  
  - Edge Cases: Invalid project ID, API errors.

- **Filtering Dartagnan Campaigns**  
  - Type: Set  
  - Role: Extracts and formats campaign details for comparison  
  - Configuration: Extracts id, campaign_name, unified_name (concatenation of name and id), creation_date, update_date, created_by, modified_by, and access_token  
  - Inputs: From Filtered Project Campaign  
  - Outputs: Connects to Existing In Braze and Not existing In Braze (merge nodes)  
  - Edge Cases: Campaigns array empty or missing fields.

---

#### 2.4 Template Comparison and Filtering

**Overview:**  
Compares Dartagnan campaigns with Braze templates to determine which templates exist in Braze and which do not, enabling update or creation actions.

**Nodes Involved:**  
- Existing In Braze  
- Not existing In Braze  
- Filter Braze vs Dartagnan  
- If campaign is modified recently  
- Sticky Note (Comparison and Sync)  

**Node Details:**  

- **Existing In Braze**  
  - Type: Merge  
  - Role: Joins Dartagnan campaigns and Braze templates on unified_name and braze_template_name to find matching templates  
  - Configuration: Join mode "keepMatches" (default combine) with merge by fields unified_name and braze_template_name  
  - Inputs: From Filtering Dartagnan Campaigns and Filtering Braze Email Template (second output)  
  - Outputs: Connects to If campaign is modified recently  
  - Edge Cases: No matches found, data type mismatches.

- **Not existing In Braze**  
  - Type: Merge  
  - Role: Finds Dartagnan campaigns that do not exist in Braze (non-matches)  
  - Configuration: Join mode "keepNonMatches" with merge by unified_name and braze_template_name  
  - Inputs: From Filtering Dartagnan Campaigns and Filtering Braze Email Template (second output)  
  - Outputs: Connects to Filter Braze vs Dartagnan  
  - Edge Cases: All templates existing or none existing.

- **Filter Braze vs Dartagnan**  
  - Type: If  
  - Role: Checks if unified_name exists (non-empty) to filter templates for creation  
  - Configuration: Condition checks existence of unified_name string  
  - Inputs: From Not existing In Braze  
  - Outputs: Connects to Dartagnan HTML & MEDIA Campagne to Create  
  - Edge Cases: Missing unified_name field.

- **If campaign is modified recently**  
  - Type: If  
  - Role: Determines if Dartagnan campaign update_date is more recent than Braze updated_at to decide if update is needed  
  - Configuration: DateTime comparison: Dartagnan update_date > Braze updated_at  
  - Inputs: From Existing In Braze  
  - Outputs: Connects to Dartagnan HTML & MEDIA To Update  
  - Edge Cases: Date format inconsistencies, missing dates.

---

#### 2.5 Template Content Processing

**Overview:**  
Fetches detailed HTML and media content from Dartagnan campaigns, processes image URLs to direct links compatible with Braze, and encodes content for API submission.

**Nodes Involved:**  
- Dartagnan HTML & MEDIA To Update  
- Dartagnan HTML & MEDIA Campagne to Create  
- Embed image in HTML  
- Embed image in HTML 1  
- Encode Content To Update  
- Encode Content to Create  
- Sticky Note (Template Processing)  

**Node Details:**  

- **Dartagnan HTML & MEDIA To Update**  
  - Type: HTTP Request  
  - Role: Retrieves full campaign details including HTML and media for templates to update  
  - Configuration: GET request to `https://app.dartagnan.io/api/public/campaigns/{{ $json.id }}` with Authorization header using Dartagnan token  
  - Inputs: From If campaign is modified recently  
  - Outputs: Connects to Embed image in HTML  
  - Edge Cases: API errors, missing campaign data.

- **Dartagnan HTML & MEDIA Campagne to Create**  
  - Type: HTTP Request  
  - Role: Retrieves full campaign details for templates to create in Braze  
  - Configuration: Same as above, GET request with campaign id  
  - Inputs: From Filter Braze vs Dartagnan  
  - Outputs: Connects to Embed image in HTML 1  
  - Edge Cases: Same as above.

- **Embed image in HTML** and **Embed image in HTML 1**  
  - Type: Code  
  - Role: JavaScript code nodes that replace image references in HTML with direct URLs from Dartagnan media assets to ensure Braze compatibility  
  - Configuration: Custom JS code that processes HTML and media objects, replacing image src, background, v:fill src, CSS background-image URLs, and CSS class references with direct URLs  
  - Inputs: From Dartagnan HTML & MEDIA To Update / Campagne to Create  
  - Outputs: Connect to Encode Content To Update / Encode Content to Create respectively  
  - Edge Cases: Missing html or medias properties, regex failures, malformed HTML.

- **Encode Content To Update** and **Encode Content to Create**  
  - Type: Set  
  - Role: JSON-stringify the processed HTML and plaintext body for API submission to Braze  
  - Configuration: Assigns encoded_html and encoded_plaintext_body fields by stringifying html and text fields respectively  
  - Inputs: From Embed image in HTML / Embed image in HTML 1  
  - Outputs: Connect to Update existing email template in Braze / Create email template  
  - Edge Cases: JSON stringify errors if content malformed.

---

#### 2.6 Template Synchronization

**Overview:**  
Pushes the processed templates to Braze by creating new templates or updating existing ones via their Content Blocks API.

**Nodes Involved:**  
- Create email template  
- Update existing email template in Braze  

**Node Details:**  

- **Create email template**  
  - Type: HTTP Request  
  - Role: Creates a new email template in Braze with the processed content  
  - Configuration: POST request to `https://{{instance_url}}/templates/email/create` with JSON body containing template_name, subject (hardcoded "Subject Line"), body (encoded_html), plaintext_body (encoded_plaintext_body), and Authorization header with Braze API key  
  - Inputs: From Encode Content to Create  
  - Outputs: None (end node)  
  - Edge Cases: API errors, invalid JSON, rate limits.

- **Update existing email template in Braze**  
  - Type: HTTP Request  
  - Role: Updates an existing Braze email template with new content from Dartagnan  
  - Configuration: POST request to `https://{{instance_url}}/templates/email/update` with JSON body containing email_template_id, template_name, subject, body, plaintext_body, and Authorization header  
  - Inputs: From Encode Content To Update  
  - Outputs: None (end node)  
  - Edge Cases: Incorrect template ID, API errors, rate limits.

---

### 3. Summary Table

| Node Name                         | Node Type               | Functional Role                                   | Input Node(s)                      | Output Node(s)                         | Sticky Note                                                                                     |
|----------------------------------|-------------------------|-------------------------------------------------|----------------------------------|--------------------------------------|------------------------------------------------------------------------------------------------|
| Every 5 minutes start             | Schedule Trigger        | Starts workflow on schedule                       | None                             | Assign Credentials                   | ## Trigger<br>Trigger is scheduled to run every 5 minutes, configurable                       |
| Assign Credentials               | Set                     | Stores API credentials                            | Every 5 minutes start            | List Available Email Template Braze, Token Request | ## Authentication Set Up<br>Obtain an access token from Dartagnan<br>Prepare credentials for both Dartagnan and Braze |
| Token Request                   | HTTP Request            | Requests OAuth2 token from Dartagnan             | Assign Credentials               | Dartagnan Project list               | ## Authentication Token<br>Obtain an access token from Dartagnan                              |
| Dartagnan Project list          | HTTP Request            | Retrieves Dartagnan projects                      | Token Request                   | Filtered Project Campaign            | ## Template Discovery<br>Retrieve project and campaign details from Dartagnan                  |
| Filtered Project Campaign       | HTTP Request            | Retrieves detailed project info with campaigns   | Dartagnan Project list          | Filtering Dartagnan Campaigns        | ## Template Discovery<br>Retrieve project and campaign details from Dartagnan                  |
| Filtering Dartagnan Campaigns   | Set                     | Extracts and formats campaign details             | Filtered Project Campaign       | Existing In Braze, Not existing In Braze | ## Template Discovery<br>Retrieve project and campaign details from Dartagnan                  |
| List Available Email Template Braze | HTTP Request        | Lists existing Braze email templates              | Assign Credentials              | Split Out                          | ## Template Discovery<br>List all existing email templates in Braze                           |
| Split Out                      | Split Out               | Splits Braze templates array into individual items | List Available Email Template Braze | Filtering Braze Email Template       | ## Template Discovery<br>List all existing email templates in Braze                           |
| Filtering Braze Email Template  | Set                     | Maps Braze template fields for comparison         | Split Out                      | Existing In Braze, Not existing In Braze | ## Template Discovery<br>List all existing email templates in Braze                           |
| Existing In Braze              | Merge                   | Matches Dartagnan campaigns with Braze templates  | Filtering Dartagnan Campaigns, Filtering Braze Email Template | If campaign is modified recently      | ## Comparison and Sync<br>Compare Dartagnan templates with existing Braze templates           |
| Not existing In Braze          | Merge                   | Finds Dartagnan campaigns not in Braze             | Filtering Dartagnan Campaigns, Filtering Braze Email Template | Filter Braze vs Dartagnan             | ## Comparison and Sync<br>Compare Dartagnan templates with existing Braze templates           |
| Filter Braze vs Dartagnan      | If                      | Filters templates for creation based on existence | Not existing In Braze           | Dartagnan HTML & MEDIA Campagne to Create | ## Comparison and Sync<br>Compare Dartagnan templates with existing Braze templates           |
| If campaign is modified recently | If                      | Checks if Dartagnan campaign is newer than Braze  | Existing In Braze              | Dartagnan HTML & MEDIA To Update     | ## Comparison and Sync<br>Compare Dartagnan templates with existing Braze templates           |
| Dartagnan HTML & MEDIA To Update | HTTP Request            | Fetches detailed Dartagnan campaign content for update | If campaign is modified recently | Embed image in HTML                 | ## Template Processing<br>Extract HTML and media from Dartagnan templates                     |
| Dartagnan HTML & MEDIA Campagne to Create | HTTP Request    | Fetches detailed Dartagnan campaign content for creation | Filter Braze vs Dartagnan       | Embed image in HTML 1               | ## Template Processing<br>Extract HTML and media from Dartagnan templates                     |
| Embed image in HTML            | Code                    | Replaces image references in HTML with direct URLs | Dartagnan HTML & MEDIA To Update | Encode Content To Update            | ## Template Processing<br>Replace image references with direct URLs                          |
| Embed image in HTML 1          | Code                    | Replaces image references in HTML with direct URLs | Dartagnan HTML & MEDIA Campagne to Create | Encode Content to Create            | ## Template Processing<br>Replace image references with direct URLs                          |
| Encode Content To Update       | Set                     | JSON-stringifies HTML and plaintext body for update | Embed image in HTML             | Update existing email template in Braze | ## Template Processing<br>Prepare templates for Braze                                       |
| Encode Content to Create       | Set                     | JSON-stringifies HTML and plaintext body for create | Embed image in HTML 1           | Create email template               | ## Template Processing<br>Prepare templates for Braze                                       |
| Update existing email template in Braze | HTTP Request        | Updates existing Braze email template              | Encode Content To Update        | None                              | ## Template Processing<br>Update existing templates in Braze                                |
| Create email template          | HTTP Request            | Creates new Braze email template                    | Encode Content to Create        | None                              | ## Template Processing<br>Create new templates in Braze                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Name: "Every 5 minutes start"  
   - Type: Schedule Trigger  
   - Configuration: Set interval to every 5 minutes (adjustable)  

2. **Create a Set node for credentials**  
   - Name: "Assign Credentials"  
   - Type: Set  
   - Add string fields:  
     - client_id: "Enter your Dartagnan client_id"  
     - client_secret: "Enter your Dartagnan client_secret"  
     - instance_url: "Enter your Braze instance_url like https://rest.fra-02.braze.eu"  
     - api_key: "Enter your Braze API key"  
   - Connect "Every 5 minutes start" → "Assign Credentials"  

3. **Create HTTP Request node to get Dartagnan token**  
   - Name: "Token Request"  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://app.dartagnan.io/oauth/v2/token`  
   - Body parameters (form or JSON):  
     - client_id: `={{ $('Assign Credentials').item.json.client_id }}`  
     - client_secret: `={{ $('Assign Credentials').item.json.client_secret }}`  
     - grant_type: "client_credentials"  
   - Connect "Assign Credentials" → "Token Request"  

4. **Create HTTP Request node to list Dartagnan projects**  
   - Name: "Dartagnan Project list"  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://app.dartagnan.io/api/public/projects`  
   - Headers: Authorization: `Bearer {{ $json.access_token }}` (token from "Token Request")  
   - Connect "Token Request" → "Dartagnan Project list"  

5. **Create HTTP Request node to get project details**  
   - Name: "Filtered Project Campaign"  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://app.dartagnan.io/api/public/projects/{{ $json.id }}`  
   - Headers: Authorization: `Bearer {{ $('Token Request').item.json.access_token }}`  
   - Connect "Dartagnan Project list" → "Filtered Project Campaign"  

6. **Create Set node to extract campaign details**  
   - Name: "Filtering Dartagnan Campaigns"  
   - Type: Set  
   - Assign fields from `$json.campaigns[0]`: id, name, unified_name (name + "-" + id), created, updated, createdBy.firstname + lastname, updatedBy.firstname + lastname, and access_token from "Token Request"  
   - Connect "Filtered Project Campaign" → "Filtering Dartagnan Campaigns"  

7. **Create HTTP Request node to list Braze email templates**  
   - Name: "List Available Email Template Braze"  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://{{ $('Assign Credentials').item.json.instance_url }}/templates/email/list`  
   - Headers: Authorization: `Bearer {{ $('Assign Credentials').item.json.api_key }}`  
   - Connect "Assign Credentials" → "List Available Email Template Braze"  

8. **Create Split Out node**  
   - Name: "Split Out"  
   - Type: Split Out  
   - Field to split: "templates"  
   - Connect "List Available Email Template Braze" → "Split Out"  

9. **Create Set node to map Braze template fields**  
   - Name: "Filtering Braze Email Template"  
   - Type: Set  
   - Assign braze_template_name, email_template_id, created_at, updated_at from input JSON  
   - Connect "Split Out" → "Filtering Braze Email Template"  

10. **Create Merge node to find existing templates**  
    - Name: "Existing In Braze"  
    - Type: Merge  
    - Mode: Combine  
    - Join Mode: Keep Matches  
    - Merge by: unified_name (left), braze_template_name (right)  
    - Connect "Filtering Dartagnan Campaigns" (main input) and "Filtering Braze Email Template" (second input) → "Existing In Braze"  

11. **Create Merge node to find new templates**  
    - Name: "Not existing In Braze"  
    - Type: Merge  
    - Mode: Combine  
    - Join Mode: Keep Non Matches  
    - Merge by: unified_name (left), braze_template_name (right)  
    - Connect "Filtering Dartagnan Campaigns" and "Filtering Braze Email Template" → "Not existing In Braze"  

12. **Create If node to filter new templates**  
    - Name: "Filter Braze vs Dartagnan"  
    - Type: If  
    - Condition: Check if unified_name exists (string operation: exists)  
    - Connect "Not existing In Braze" → "Filter Braze vs Dartagnan"  

13. **Create If node to check if campaign modified recently**  
    - Name: "If campaign is modified recently"  
    - Type: If  
    - Condition: Dartagnan update_date > Braze updated_at (dateTime after)  
    - Connect "Existing In Braze" → "If campaign is modified recently"  

14. **Create HTTP Request node to fetch Dartagnan campaign content for update**  
    - Name: "Dartagnan HTML & MEDIA To Update"  
    - Type: HTTP Request  
    - Method: GET  
    - URL: `https://app.dartagnan.io/api/public/campaigns/{{ $json.id }}`  
    - Headers: Authorization: `Bearer {{ $('Token Request').item.json.access_token }}`  
    - Connect "If campaign is modified recently" → "Dartagnan HTML & MEDIA To Update"  

15. **Create HTTP Request node to fetch Dartagnan campaign content for create**  
    - Name: "Dartagnan HTML & MEDIA Campagne to Create"  
    - Type: HTTP Request  
    - Method: GET  
    - URL: `https://app.dartagnan.io/api/public/campaigns/{{ $json.id }}`  
    - Headers: Authorization: `Bearer {{ $('Token Request').item.json.access_token }}`  
    - Connect "Filter Braze vs Dartagnan" → "Dartagnan HTML & MEDIA Campagne to Create"  

16. **Create Code node to embed images in HTML for update**  
    - Name: "Embed image in HTML"  
    - Type: Code  
    - Paste the provided JavaScript code that replaces image references with direct URLs  
    - Connect "Dartagnan HTML & MEDIA To Update" → "Embed image in HTML"  

17. **Create Code node to embed images in HTML for create**  
    - Name: "Embed image in HTML 1"  
    - Type: Code  
    - Same JavaScript code as above  
    - Connect "Dartagnan HTML & MEDIA Campagne to Create" → "Embed image in HTML 1"  

18. **Create Set node to encode content for update**  
    - Name: "Encode Content To Update"  
    - Type: Set  
    - Assign encoded_html = JSON.stringify(html), encoded_plaintext_body = JSON.stringify(text)  
    - Connect "Embed image in HTML" → "Encode Content To Update"  

19. **Create Set node to encode content for create**  
    - Name: "Encode Content to Create"  
    - Type: Set  
    - Same assignments as above  
    - Connect "Embed image in HTML 1" → "Encode Content to Create"  

20. **Create HTTP Request node to update existing Braze template**  
    - Name: "Update existing email template in Braze"  
    - Type: HTTP Request  
    - Method: POST  
    - URL: `https://{{ $('Assign Credentials').item.json.instance_url }}/templates/email/update`  
    - Headers: Authorization: Bearer Braze API key, Content-Type: application/json  
    - Body JSON: email_template_id, template_name, subject ("Subject Line"), body (encoded_html), plaintext_body (encoded_plaintext_body)  
    - Connect "Encode Content To Update" → "Update existing email template in Braze"  

21. **Create HTTP Request node to create new Braze template**  
    - Name: "Create email template"  
    - Type: HTTP Request  
    - Method: POST  
    - URL: `https://{{ $('Assign Credentials').item.json.instance_url }}/templates/email/create`  
    - Headers: Authorization: Bearer Braze API key, Content-Type: application/json  
    - Body JSON: template_name, subject ("Subject Line"), body (encoded_html), plaintext_body (encoded_plaintext_body)  
    - Connect "Encode Content to Create" → "Create email template"  

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| The workflow is designed to run every 5 minutes by default but can be adjusted to respect API rate limits | Sticky Note near "Every 5 minutes start" node                                                    |
| Authentication requires Dartagnan client_id and client_secret, and Braze instance URL and API key        | Sticky Notes near "Assign Credentials" and "Token Request" nodes                                |
| The JavaScript code nodes for embedding images handle multiple HTML and CSS patterns for image URLs      | Embedded in "Embed image in HTML" and "Embed image in HTML 1" nodes                             |
| Braze API endpoints used: `/templates/email/list`, `/templates/email/create`, `/templates/email/update` | Braze REST API documentation recommended for further customization                              |
| Initial sync creates all templates; subsequent runs update only changed templates                         | Workflow description and "If campaign is modified recently" node logic                          |
| Error handling and notifications are not explicitly implemented but can be added as customization        | Workflow description under Customization Options                                                |

---

This document provides a detailed, structured reference for understanding, reproducing, and maintaining the "Sync Dartagnan Email Templates to Braze" workflow in n8n. It covers all nodes, their configurations, logical flow, and integration points.
Automate Zoom User Onboarding with OAuth Token Management and Data Tables

https://n8nworkflows.xyz/workflows/automate-zoom-user-onboarding-with-oauth-token-management-and-data-tables-10747


# Automate Zoom User Onboarding with OAuth Token Management and Data Tables

### 1. Workflow Overview

This workflow automates Zoom user onboarding by managing OAuth token refresh cycles and creating new Zoom users via the Zoom API. It is designed to handle Zoom’s short-lived access tokens by utilizing a refresh token stored in an n8n Data Table. The workflow logic is divided into the following functional blocks:

- **1.1 Trigger and Initial Data Setup:** Manual trigger initiates the workflow and sets user data for the new Zoom account.
- **1.2 Token Retrieval and Management:** Retrieves stored OAuth tokens from a Data Table, selects the most recent token, and requests a refreshed access token using the stored refresh token.
- **1.3 Token Storage:** Inserts the newly refreshed tokens back into the Data Table for future use.
- **1.4 User Creation:** Generates a random password, merges it with user data, and calls the Zoom API to create a new Zoom user.
- **1.5 Conditional Handling:** Checks whether the user creation request was successful.
- **1.6 First-Time Setup Subprocess:** A separate node to generate the initial OAuth tokens manually before the main workflow can be used automatically.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and Initial Data Setup

- **Overview:** Starts the workflow manually and sets up the new user’s first name, last name, and email.
- **Nodes Involved:**  
  - When clicking ‘Execute workflow’  
  - Data  
  - Get row(s)  

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Starts the workflow on-demand.  
    - Configuration: No parameters; simple manual trigger.  
    - Inputs: None  
    - Outputs: Connects to "Data" node.  
    - Potential Failures: None typical; user must manually execute.

  - **Data**  
    - Type: Set  
    - Role: Defines static data for new Zoom user creation.  
    - Configuration: Hardcoded fields for first_name, last_name, and email with placeholder values "XXX" to be replaced by user.  
    - Expressions: Static string assignments; no dynamic expressions.  
    - Inputs: From manual trigger  
    - Outputs: To "Get row(s)" node.  
    - Edge Cases: If data is incomplete or placeholders not replaced, Zoom API may reject user creation.

  - **Get row(s)**  
    - Type: Data Table  
    - Role: Retrieves all stored OAuth tokens from the “Zoom Access Token” Data Table.  
    - Configuration: Operation "get", return all rows, matching all conditions (none specified, so all rows returned).  
    - Inputs: From "Data" node  
    - Outputs: To "Sort" node.  
    - Potential Failures: Data Table missing or inaccessible, permission issues.

#### 2.2 Token Retrieval and Management

- **Overview:** Sorts tokens, limits to the most recent token, and uses the refresh token to get a new access token from Zoom OAuth API.
- **Nodes Involved:**  
  - Sort  
  - Limit  
  - Get new token  

- **Node Details:**

  - **Sort**  
    - Type: Sort  
    - Role: Sorts token entries descending by `id` to get the latest token.  
    - Configuration: Sort by field "id" descending.  
    - Inputs: From "Get row(s)"  
    - Outputs: To "Limit" node.  
    - Edge Cases: Empty token list results in no token available.

  - **Limit**  
    - Type: Limit  
    - Role: Limits output to the single most recent token.  
    - Configuration: Default limit of 1 (implied).  
    - Inputs: From "Sort"  
    - Outputs: To "Get new token".  
    - Edge Cases: No tokens, no output.

  - **Get new token**  
    - Type: HTTP Request  
    - Role: Requests Zoom OAuth token endpoint to refresh access token using stored refresh token.  
    - Configuration:  
      - Method: POST  
      - URL: https://zoom.us/oauth/token  
      - Content-Type: application/x-www-form-urlencoded  
      - Body parameters: grant_type=refresh_token, refresh_token from JSON input (`{{$json.refreshToken}}`)  
      - Headers: Authorization with Basic Auth (Base64(client_id:client_secret)) — placeholder "XXX" to be replaced.  
    - Inputs: From "Limit" node (provides refresh token)  
    - Outputs: To "Insert new token" node.  
    - Edge Cases:  
      - Invalid refresh token (expired or revoked) causing 401 Unauthorized.  
      - Network timeouts or Zoom API errors.  
      - Missing or incorrect Authorization header.  
      - Rate limiting by Zoom API.

#### 2.3 Token Storage

- **Overview:** Saves the new access and refresh tokens into the Data Table for future token refreshes.
- **Nodes Involved:**  
  - Insert new token  

- **Node Details:**

  - **Insert new token**  
    - Type: Data Table  
    - Role: Inserts a new row with the refreshed access_token and refresh_token.  
    - Configuration:  
      - Mapping: accessToken ← `{{$json.access_token}}`, refreshToken ← `{{$json.refresh_token}}`  
      - Data Table: “Zoom Access Token” identified by ID `yLOjlMChh6CiymeJ`  
    - Inputs: From "Get new token" (provides new tokens)  
    - Outputs: To "Generate random Password" and "Merge" nodes (parallel outputs).  
    - Edge Cases: Data Table unavailable, write permission errors.

#### 2.4 User Creation

- **Overview:** Generates a random password, merges it with user data, and sends a request to Zoom to create a new user.
- **Nodes Involved:**  
  - Generate random Password  
  - Merge  
  - Create new user  

- **Node Details:**

  - **Generate random Password**  
    - Type: Set  
    - Role: Creates a random 15-character password with appended special character.  
    - Configuration:  
      - Expression generates a random alphanumeric string of length 15  
      - Adds one random special character from set !, @, #, $, %, &, *  
      - Output variable: `password`  
    - Inputs: From "Insert new token"  
    - Outputs: To "Merge" node.  
    - Edge Cases: Low entropy if random seed is weak; ensure randomness is sufficient.

  - **Merge**  
    - Type: Merge  
    - Role: Combines user data (from "Data" node) with generated password.  
    - Configuration: Mode "combine" with "combineAll".  
    - Inputs:  
      - From "Generate random Password" (password data)  
      - From "Insert new token" (user data)  
    - Outputs: To "Create new user".  
    - Edge Cases: Mismatched input data counts may cause data misalignment.

  - **Create new user**  
    - Type: HTTP Request  
    - Role: Calls Zoom API to create a new user with supplied data and password.  
    - Configuration:  
      - URL: https://api.zoom.us/v2/users  
      - Method: POST  
      - Body: JSON including action "create", user_info with email, type=2 (licensed user), first_name, last_name, password  
      - Headers: Authorization Bearer token from `{{$json.accessToken}}`, Content-Type application/json  
    - Inputs: From "Merge"  
    - Outputs: To "If" node.  
    - Edge Cases:  
      - Invalid or expired access token causing 401  
      - User data invalid or email already exists causing 400/409 errors  
      - Network or Zoom API downtime  
      - Rate limiting or quota exceeded  
      - Password policy violations

#### 2.5 Conditional Handling

- **Overview:** Checks if user creation succeeded based on the presence of `id` in the response.
- **Nodes Involved:**  
  - If  

- **Node Details:**

  - **If**  
    - Type: If  
    - Role: Tests if response JSON contains a valid `id` field indicating success.  
    - Configuration: Condition checks existence of `id` field in response JSON strictly.  
    - Inputs: From "Create new user"  
    - Outputs: No connected outputs (empty array in main branch).  
    - Edge Cases:  
      - API success without `id` field (unlikely but possible)  
      - Errors or empty responses cause condition to fail

#### 2.6 First-Time Setup Subprocess

- **Overview:** A separate node to obtain the initial access and refresh tokens via authorization code flow, executed only once.
- **Nodes Involved:**  
  - Zoom First Access Token  
  - Insert token  

- **Node Details:**

  - **Zoom First Access Token**  
    - Type: HTTP Request  
    - Role: Uses authorization_code grant to obtain initial tokens.  
    - Configuration:  
      - URL: https://zoom.us/oauth/token  
      - Method: POST  
      - Body form-urlencoded parameters: grant_type=authorization_code, code (hardcoded authorization code), redirect_uri pointing to n8n OAuth callback  
      - Headers: Authorization with Basic Auth (Base64(client_id:client_secret)) placeholder "XXX"  
    - Inputs: None (manual run)  
    - Outputs: To "Insert token" node.  
    - Edge Cases:  
      - Authorization code expired or already used  
      - Incorrect redirect_uri or headers  
      - Network issues, API errors

  - **Insert token**  
    - Type: Data Table  
    - Role: Inserts initial tokens into Data Table.  
    - Configuration: Same as "Insert new token".  
    - Inputs: From "Zoom First Access Token"  
    - Outputs: None  
    - Edge Cases: Same as "Insert new token".

---

### 3. Summary Table

| Node Name                    | Node Type           | Functional Role                                  | Input Node(s)                         | Output Node(s)                  | Sticky Note                                                                                                          |
|------------------------------|---------------------|-------------------------------------------------|-------------------------------------|--------------------------------|----------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger      | Starts workflow manually                         | None                                | Data                           |                                                                                                                      |
| Data                         | Set                 | Sets new user first name, last name, and email | When clicking ‘Execute workflow’    | Get row(s)                     | First name, Last name about the user to create                                                                       |
| Get row(s)                   | Data Table          | Retrieves stored Zoom OAuth tokens               | Data                                | Sort                           | Get only the recent token added                                                                                      |
| Sort                         | Sort                | Sorts tokens descending by id                     | Get row(s)                         | Limit                          | Get only the recent token added                                                                                      |
| Limit                        | Limit               | Limits to the most recent token                   | Sort                               | Get new token                  | Get only the recent token added                                                                                      |
| Get new token                | HTTP Request        | Refreshes access token using refresh token       | Limit                              | Insert new token               |                                                                                                                      |
| Insert new token             | Data Table          | Inserts refreshed tokens into Data Table         | Get new token                      | Generate random Password, Merge | Add the new token and refresh_token in the table                                                                     |
| Generate random Password     | Set                 | Generates random password for new user            | Insert new token                   | Merge                         |                                                                                                                      |
| Merge                       | Merge               | Combines user data with generated password        | Generate random Password, Insert new token | Create new user               |                                                                                                                      |
| Create new user             | HTTP Request        | Creates new Zoom user via Zoom API                 | Merge                             | If                            | Create new Zoom user                                                                                                  |
| If                          | If                  | Checks if user creation succeeded                  | Create new user                   | None                          | If there are seats available, an email to new user                                                                   |
| Zoom First Access Token      | HTTP Request        | One-time: obtains first access and refresh tokens | Manual run (not connected)         | Insert token                   | Generate your first access token                                                                                      |
| Insert token                | Data Table          | Inserts initial tokens into Data Table             | Zoom First Access Token            | None                          | Generate your first access token                                                                                      |
| Sticky Note                 | Sticky Note         | Provides instructions and notes                     | N/A                               | N/A                           | Multiple sticky notes provide step-by-step instructions and workflow explanations (see detailed notes below)         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Node Type: Manual Trigger  
   - Purpose: To start the workflow manually.

2. **Create Set Node "Data"**  
   - Node Type: Set  
   - Purpose: Define user info for Zoom user creation.  
   - Parameters:  
     - first_name: Replace "XXX" with the new user’s first name  
     - last_name: Replace "XXX" with the new user’s last name  
     - email: Replace "XXX" with the new user’s email address  
   - Connect manual trigger output to this node.

3. **Create Data Table Node "Get row(s)"**  
   - Node Type: Data Table  
   - Purpose: Retrieve all stored OAuth tokens.  
   - Parameters:  
     - Operation: get  
     - Return All: true  
     - Data Table: Select or create a Data Table with columns `accessToken` and `refreshToken` (see setup below).  
   - Connect "Data" node output here.

4. **Create Sort Node**  
   - Node Type: Sort  
   - Purpose: Sort tokens descending by `id` to get newest tokens first.  
   - Parameters: Sort by field `id` descending.  
   - Connect "Get row(s)" output here.

5. **Create Limit Node**  
   - Node Type: Limit  
   - Purpose: Limit results to 1 (the most recent token).  
   - Parameters: Defaults suffice (limit=1).  
   - Connect "Sort" output here.

6. **Create HTTP Request Node "Get new token"**  
   - Node Type: HTTP Request  
   - Purpose: Refresh access token using the refresh token.  
   - Parameters:  
     - URL: https://zoom.us/oauth/token  
     - Method: POST  
     - Content-Type: application/x-www-form-urlencoded  
     - Body Parameters: grant_type=refresh_token, refresh_token from input `{{$json.refreshToken}}`  
     - Headers: Authorization: Basic <BASE64_ENCODED_CLIENTID_CLIENTSECRET> (replace "XXX" with your encoded value)  
   - Connect "Limit" output here.

7. **Create Data Table Node "Insert new token"**  
   - Node Type: Data Table  
   - Purpose: Store refreshed tokens for future use.  
   - Parameters:  
     - Columns: accessToken = `{{$json.access_token}}`, refreshToken = `{{$json.refresh_token}}`  
     - Target Data Table: Same as in step 3.  
   - Connect "Get new token" output here.

8. **Create Set Node "Generate random Password"**  
   - Node Type: Set  
   - Purpose: Generate a random password for the new Zoom user.  
   - Parameters:  
     - Expression:  
       `={{ Array.from({length: 15}, () => Math.random().toString(36).charAt(2)).join('') + ['!', '@', '#', '$', '%', '&', '*'][Math.floor(Math.random() * 7)] }}`  
     - Output variable: `password`  
   - Connect "Insert new token" output here.

9. **Create Merge Node**  
   - Node Type: Merge  
   - Purpose: Combine user data and generated password into one data object.  
   - Parameters: Mode “Combine”, combine all.  
   - Connect outputs:  
     - From "Generate random Password" to first input  
     - From "Insert new token" to second input  
   - Connect merge output to next node.

10. **Create HTTP Request Node "Create new user"**  
    - Node Type: HTTP Request  
    - Purpose: Create Zoom user using Zoom API.  
    - Parameters:  
      - URL: https://api.zoom.us/v2/users  
      - Method: POST  
      - Headers:  
        - Authorization: Bearer `{{$json.accessToken}}`  
        - Content-Type: application/json  
      - Body (JSON):  
        ```json
        {
          "action": "create",
          "user_info": {
            "email": "{{ $('Data').item.json.email }}",
            "type": 2,
            "first_name": "{{ $('Data').item.json.first_name }}",
            "last_name": "{{ $('Data').item.json.last_name }}",
            "password": "{{ $json.password }}"
          }
        }
        ```  
      - Send body as JSON  
    - Connect "Merge" output here.

11. **Create If Node**  
    - Node Type: If  
    - Purpose: Check if user creation succeeded by verifying presence of `id` field in response.  
    - Parameters:  
      - Condition: `{{$json.id}}` exists (string operator "exists")  
    - Connect "Create new user" output here.

12. **Create Manual HTTP Request Node "Zoom First Access Token"** (One-time use)  
    - Node Type: HTTP Request  
    - Purpose: Get initial access and refresh tokens via authorization code.  
    - Parameters:  
      - URL: https://zoom.us/oauth/token  
      - Method: POST  
      - Content-Type: application/x-www-form-urlencoded  
      - Body Parameters:  
        - grant_type=authorization_code  
        - code=<authorization_code> (replace with your real code)  
        - redirect_uri=https://oauth.n8n.cloud/oauth2/callback  
      - Headers: Authorization: Basic <BASE64_ENCODED_CLIENTID_CLIENTSECRET>  
    - Execute manually once to populate initial tokens.

13. **Create Data Table Node "Insert token"** (for first access token)  
    - Node Type: Data Table  
    - Purpose: Insert the initial tokens into Data Table.  
    - Parameters: Same as "Insert new token".  
    - Connect "Zoom First Access Token" output here.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Context or Link                                                                                                                             |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| This workflow automates the management of Zoom OAuth tokens and creation of new Zoom users through Zoom API. It handles Zoom’s short-lived (1 hour) access tokens by refreshing them using a long-lived (90 days) refresh token stored in an n8n Data Table.                                                                                                                                                                                                                                                | Sticky Note4 (top-left near workflow start)                                                                                               |
| Setup Steps: 1. Create n8n Data Table with `accessToken` and `refreshToken` fields. 2. Create Zoom OAuth App (standard OAuth, not Server-to-Server). 3. Base64 encode client_id and client_secret and set in Authorization header of HTTP nodes. 4. Run “Zoom First Access Token” node manually once to generate initial tokens. 5. Set user data in "Data" node with real user info.                                                                                                                                                     | Sticky Note12                                                                                                                             |
| Step 1: Create Data Table with two fields: accessToken and refreshToken.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Sticky Note1                                                                                                                              |
| Step 2: Create Zoom OAuth App and get account_id. Insert Base64 encoded client_id:client_secret into Authorization headers.                                                                                                                                                                                                                                                                                                                                                                                                                                | Sticky Note2                                                                                                                              |
| Step 3: Run “Zoom First Access Token” node once manually to generate initial tokens.                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Sticky Note3                                                                                                                              |
| Step 4: Set user data in "Data" node.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Sticky Note6                                                                                                                              |
| The workflow automatically refreshes tokens, stores new tokens, generates user passwords, and creates new Zoom users with invitation emails sent if seats are available.                                                                                                                                                                                                                                                                                                                                                                                    | Sticky Note11                                                                                                                             |
| Generate your first access token manually by running “Zoom First Access Token” node and inserting tokens into the Data Table before running the main workflow.                                                                                                                                                                                                                                                                                                                                                                                            | Sticky Note                                                                                                                               |
| First name, Last name about the user to create must be properly set in the "Data" node before execution.                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Sticky Note5                                                                                                                              |
| Zoom API documentation for OAuth token management and user creation can be found at https://marketplace.zoom.us/docs/api-reference/zoom-api/oauth/.                                                                                                                                                                                                                                                                                                                                                                                                        | External resource (not linked directly but recommended)                                                                                   |

---

**Disclaimer:** This document describes a workflow created exclusively using n8n automation software. All data and processes comply with current content policies and use only legal, public, and authorized data sources.
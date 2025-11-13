Build Production-Ready User Authentication with Airtable and JWT

https://n8nworkflows.xyz/workflows/build-production-ready-user-authentication-with-airtable-and-jwt-4444


# Build Production-Ready User Authentication with Airtable and JWT

### 1. Workflow Overview

This workflow implements a **production-ready user authentication service** using Airtable as the user database and cryptographic hashing for password security. It supports four primary user operations: **sign up**, **sign in**, **get user details**, and **update user details**. The workflow is designed for serverless API usage via webhooks, suitable for integrating user authentication into various web or mobile applications.

The workflow is logically divided into the following blocks:

- **1.1 User Sign-Up Process:** Validates new user emails, hashes passwords, stores user data in Airtable, and responds accordingly.
- **1.2 User Sign-In Process:** Validates user credentials by comparing submitted passwords with stored hashed passwords and returns success or error responses.
- **1.3 User Details Retrieval:** Fetches current user details from Airtable upon request.
- **1.4 User Details Update:** Updates user data in Airtable with new details submitted by the user.
- **1.5 Auxiliary Nodes:** Includes nodes for setting data, responding to webhooks, and utility nodes such as cryptographic hashing.

---

### 2. Block-by-Block Analysis

#### 2.1 User Sign-Up Process

**Overview:**  
Handles new user registration by checking if the email already exists in Airtable, hashing the password if not, adding the new user record, and responding to the client.

**Nodes Involved:**  
- `request to sign up` (Webhook)  
- `Check if email in database` (Airtable)  
- `If email in database` (If)  
- `respond with email already in use` (Respond to Webhook)  
- `hash password` (Crypto)  
- `add record to database` (Airtable)  
- `respond with sign up successful` (Respond to Webhook)

**Node Details:**

- **request to sign up**  
  - *Type:* Webhook  
  - *Role:* Entry point for user sign-up HTTP requests.  
  - *Config:* Public webhook with unique ID; expects sign-up data (email, password).  
  - *Input/Output:* Output to "Check if email in database".  
  - *Failure Modes:* Webhook misconfiguration, malformed input data.

- **Check if email in database**  
  - *Type:* Airtable (Query)  
  - *Role:* Queries Airtable for existing user records matching the submitted email.  
  - *Config:* Airtable credentials and base/table configured to search by email field.  
  - *Input/Output:* Input from webhook; output to conditional "If email in database".  
  - *Failure Modes:* Airtable API errors, network issues.

- **If email in database**  
  - *Type:* If (Boolean condition)  
  - *Role:* Checks if the queried email exists (i.e., query returned records).  
  - *Config:* Condition checks if Airtable query result is non-empty.  
  - *Input/Output:* Yes branch to "respond with email already in use", No branch to "hash password".  
  - *Failure Modes:* Expression errors if query structure changes.

- **respond with email already in use**  
  - *Type:* Respond to Webhook  
  - *Role:* Sends HTTP response indicating the email is already registered.  
  - *Config:* HTTP status and response message set appropriately (e.g., 409 Conflict).  
  - *Input/Output:* Terminal node for this branch.  
  - *Failure Modes:* Webhook response errors.

- **hash password**  
  - *Type:* Crypto  
  - *Role:* Hashes the plaintext password securely.  
  - *Config:* Uses a strong hashing algorithm (e.g., bcrypt or SHA-256 with salt).  
  - *Input/Output:* Input from "If email in database" No branch; output to "add record to database".  
  - *Failure Modes:* Crypto library errors, invalid input.

- **add record to database**  
  - *Type:* Airtable (Create)  
  - *Role:* Inserts a new user record with email and hashed password into Airtable.  
  - *Config:* Airtable credentials with write access; mapping of fields for user data.  
  - *Input/Output:* Input from "hash password"; output to "respond with sign up successful".  
  - *Failure Modes:* Airtable write failures, permission errors.

- **respond with sign up successful**  
  - *Type:* Respond to Webhook  
  - *Role:* Sends success response to client confirming registration.  
  - *Config:* HTTP 200 or 201 status with confirmation message.  
  - *Input/Output:* Terminal node for this branch.

---

#### 2.2 User Sign-In Process

**Overview:**  
Manages user login by verifying that the user exists, hashing the submitted password, comparing it with the stored hash, and returning success or error messages.

**Nodes Involved:**  
- `request to sign in` (Webhook)  
- `check if user exists` (Airtable)  
- `If user exists` (If)  
- `respond with wrong email submitted` (Respond to Webhook)  
- `hash submitted password` (Crypto)  
- `If passwords match` (If)  
- `respond with successful login` (Respond to Webhook)  
- `respond with wrong password submitted` (Respond to Webhook)

**Node Details:**

- **request to sign in**  
  - *Type:* Webhook  
  - *Role:* Entry point for user login HTTP requests.  
  - *Config:* Public webhook with unique ID expecting email and password.  
  - *Input/Output:* Output to "check if user exists".  
  - *Failure Modes:* Misconfigured webhook, invalid request data.

- **check if user exists**  
  - *Type:* Airtable (Query)  
  - *Role:* Queries Airtable for user by email.  
  - *Config:* Airtable credentials and base/table; search by email.  
  - *Input/Output:* Input from webhook; output to "If user exists".  
  - *Failure Modes:* API errors, network issues.

- **If user exists**  
  - *Type:* If  
  - *Role:* Branches based on whether the user was found.  
  - *Config:* Condition checks if Airtable returned a user record.  
  - *Input/Output:* Yes branch to "hash submitted password", No branch to "respond with wrong email submitted".  
  - *Failure Modes:* Expression failures if data format changes.

- **respond with wrong email submitted**  
  - *Type:* Respond to Webhook  
  - *Role:* Sends error response indicating email not found.  
  - *Config:* HTTP 401 or 404 with error message.  
  - *Input/Output:* Terminal node.

- **hash submitted password**  
  - *Type:* Crypto  
  - *Role:* Hashes the submitted password for comparison.  
  - *Config:* Same hashing algorithm as sign-up.  
  - *Input/Output:* Input from "If user exists" Yes branch; output to "If passwords match".  
  - *Failure Modes:* Crypto errors, invalid input.

- **If passwords match**  
  - *Type:* If  
  - *Role:* Compares hashed submitted password with stored hash.  
  - *Config:* Boolean expression comparing hashed values.  
  - *Input/Output:* Yes branch to "respond with successful login", No branch to "respond with wrong password submitted".  
  - *Failure Modes:* Expression errors.

- **respond with successful login**  
  - *Type:* Respond to Webhook  
  - *Role:* Returns success response on correct credentials.  
  - *Config:* HTTP 200 with login confirmation and possibly token (not detailed).  
  - *Input/Output:* Terminal node.

- **respond with wrong password submitted**  
  - *Type:* Respond to Webhook  
  - *Role:* Sends error response for incorrect password.  
  - *Config:* HTTP 401 Unauthorized with error message.  
  - *Input/Output:* Terminal node.

---

#### 2.3 User Details Retrieval

**Overview:**  
Processes requests to fetch current user details from Airtable and returns them.

**Nodes Involved:**  
- `request to get user details` (Webhook)  
- `get current details` (Airtable)  
- `Specify Current Details` (Set)

**Node Details:**

- **request to get user details**  
  - *Type:* Webhook  
  - *Role:* Entry point for retrieving user info.  
  - *Config:* Public webhook expecting user identifier.  
  - *Input/Output:* Output to "get current details".  
  - *Failure Modes:* Missing parameters, webhook errors.

- **get current details**  
  - *Type:* Airtable (Query)  
  - *Role:* Queries Airtable to fetch current user data by ID or email.  
  - *Config:* Airtable read access, filtering by user identifier.  
  - *Input/Output:* Output to "Specify Current Details".  
  - *Failure Modes:* Airtable API errors, no records found.

- **Specify Current Details**  
  - *Type:* Set  
  - *Role:* Formats or selects specific fields to return to client.  
  - *Config:* Sets structured output data (e.g., user name, email).  
  - *Input/Output:* Final output to webhook response (not explicitly shown).  
  - *Failure Modes:* Misconfiguration of fields.

---

#### 2.4 User Details Update

**Overview:**  
Accepts requests to update user details, writes updated info to Airtable, and confirms the update.

**Nodes Involved:**  
- `request to update user details` (Webhook)  
- `update with new details` (Airtable)  
- `Specify New Details` (Set)

**Node Details:**

- **request to update user details**  
  - *Type:* Webhook  
  - *Role:* Entry point for user detail update requests.  
  - *Config:* Public webhook expecting updated user info and identifier.  
  - *Input/Output:* Output to "update with new details".  
  - *Failure Modes:* Invalid input, webhook errors.

- **update with new details**  
  - *Type:* Airtable (Update)  
  - *Role:* Updates user record with new data in Airtable.  
  - *Config:* Airtable credentials with write permissions; mapping updated fields.  
  - *Input/Output:* Output to "Specify New Details".  
  - *Failure Modes:* API errors, permission issues.

- **Specify New Details**  
  - *Type:* Set  
  - *Role:* Prepares confirmation or response data after update.  
  - *Config:* Sets output data for confirmation message.  
  - *Input/Output:* Final node for this branch.  
  - *Failure Modes:* Misconfiguration.

---

#### 2.5 Auxiliary Nodes (Sticky Notes)

- Nodes named `Sticky Note2`, `Sticky Note3`, and `Sticky Note6` appear in the workflow but contain no content. They likely serve as placeholders or markers for future notes or documentation.

---

### 3. Summary Table

| Node Name                    | Node Type          | Functional Role                      | Input Node(s)               | Output Node(s)                        | Sticky Note |
|------------------------------|--------------------|------------------------------------|-----------------------------|-------------------------------------|-------------|
| request to sign up            | Webhook            | Entry point for user sign-up       |                             | Check if email in database           |             |
| Check if email in database    | Airtable           | Query user email existence          | request to sign up           | If email in database                 |             |
| If email in database          | If                 | Branch if email exists              | Check if email in database   | respond with email already in use, hash password |             |
| respond with email already in use | Respond to Webhook | Sign-up failure response           | If email in database (yes)   |                                     |             |
| hash password                | Crypto             | Hash user password                  | If email in database (no)    | add record to database               |             |
| add record to database        | Airtable           | Create new user record              | hash password               | respond with sign up successful     |             |
| respond with sign up successful | Respond to Webhook | Sign-up success response            | add record to database       |                                     |             |
| request to sign in            | Webhook            | Entry point for user sign-in       |                             | check if user exists                 |             |
| check if user exists          | Airtable           | Query user by email                 | request to sign in           | If user exists                      |             |
| If user exists                | If                 | Branch if user found                | check if user exists         | hash submitted password, respond with wrong email submitted |             |
| respond with wrong email submitted | Respond to Webhook | Sign-in failure for unknown email | If user exists (no)          |                                     |             |
| hash submitted password       | Crypto             | Hash submitted password             | If user exists (yes)         | If passwords match                  |             |
| If passwords match            | If                 | Compare stored and submitted hashes | hash submitted password      | respond with successful login, respond with wrong password submitted |             |
| respond with successful login | Respond to Webhook | Sign-in success response            | If passwords match (yes)     |                                     |             |
| respond with wrong password submitted | Respond to Webhook | Sign-in failure for wrong password | If passwords match (no)      |                                     |             |
| request to get user details   | Webhook            | Entry point for fetching user data |                             | get current details                 |             |
| get current details           | Airtable           | Query current user data             | request to get user details  | Specify Current Details             |             |
| Specify Current Details       | Set                | Format current user data            | get current details          |                                     |             |
| request to update user details| Webhook            | Entry point for updating user data |                             | update with new details             |             |
| update with new details       | Airtable           | Update user record                  | request to update user details | Specify New Details               |             |
| Specify New Details           | Set                | Format update confirmation          | update with new details      |                                     |             |
| Sticky Note2                 | Sticky Note        | Placeholder note                   |                             |                                     |             |
| Sticky Note3                 | Sticky Note        | Placeholder note                   |                             |                                     |             |
| Sticky Note6                 | Sticky Note        | Placeholder note                   |                             |                                     |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node:**  
   - Name: `request to sign up`  
   - Type: Webhook (HTTP POST)  
   - Configure to accept JSON with user email and password.

2. **Create Airtable Node:**  
   - Name: `Check if email in database`  
   - Type: Airtable (List Records)  
   - Configure with Airtable credentials, base, and table containing users.  
   - Filter records by the submitted email from webhook data.

3. **Create If Node:**  
   - Name: `If email in database`  
   - Type: If  
   - Condition: Check if the Airtable query from step 2 returns any records.  
   - If true, connect to step 4; if false, connect to step 5.

4. **Create Respond to Webhook Node:**  
   - Name: `respond with email already in use`  
   - Type: Respond to Webhook  
   - Configure to send HTTP 409 Conflict and message "Email already registered".

5. **Create Crypto Node:**  
   - Name: `hash password`  
   - Type: Crypto  
   - Configure to hash the password from the webhook using a secure algorithm (e.g., bcrypt or SHA-256 with salt).

6. **Create Airtable Node:**  
   - Name: `add record to database`  
   - Type: Airtable (Create Record)  
   - Configure to insert a new user record with email and hashed password.

7. **Create Respond to Webhook Node:**  
   - Name: `respond with sign up successful`  
   - Type: Respond to Webhook  
   - Configure to send HTTP 201 Created with success message.

8. **Connect nodes:**  
   - `request to sign up` → `Check if email in database` → `If email in database`  
   - `If email in database` (true) → `respond with email already in use`  
   - `If email in database` (false) → `hash password` → `add record to database` → `respond with sign up successful`

9. **Create Webhook Node:**  
   - Name: `request to sign in`  
   - Type: Webhook (POST)  
   - Configure to accept JSON with email and password.

10. **Create Airtable Node:**  
    - Name: `check if user exists`  
    - Type: Airtable (List Records)  
    - Configure to query user by submitted email.

11. **Create If Node:**  
    - Name: `If user exists`  
    - Condition: Check if user record exists from previous query.

12. **Create Respond to Webhook Node:**  
    - Name: `respond with wrong email submitted`  
    - Configure HTTP 401 Unauthorized with message "Email not found".

13. **Create Crypto Node:**  
    - Name: `hash submitted password`  
    - Configure to hash the submitted password.

14. **Create If Node:**  
    - Name: `If passwords match`  
    - Condition: Compare hashed submitted password with stored hashed password from Airtable.

15. **Create Respond to Webhook Nodes:**  
    - Name: `respond with successful login` — HTTP 200 with success message.  
    - Name: `respond with wrong password submitted` — HTTP 401 Unauthorized with error message.

16. **Connect nodes:**  
    - `request to sign in` → `check if user exists` → `If user exists`  
    - `If user exists` (false) → `respond with wrong email submitted`  
    - `If user exists` (true) → `hash submitted password` → `If passwords match`  
    - `If passwords match` (true) → `respond with successful login`  
    - `If passwords match` (false) → `respond with wrong password submitted`

17. **Create Webhook Node:**  
    - Name: `request to get user details`  
    - Configure to accept user identifier.

18. **Create Airtable Node:**  
    - Name: `get current details`  
    - Query Airtable for user data by identifier.

19. **Create Set Node:**  
    - Name: `Specify Current Details`  
    - Set the output data structure for user details response.

20. **Connect nodes:**  
    - `request to get user details` → `get current details` → `Specify Current Details`

21. **Create Webhook Node:**  
    - Name: `request to update user details`  
    - Configure to accept user identifier and new data.

22. **Create Airtable Node:**  
    - Name: `update with new details`  
    - Configure to update the user record with new details.

23. **Create Set Node:**  
    - Name: `Specify New Details`  
    - Format confirmation or updated data response.

24. **Connect nodes:**  
    - `request to update user details` → `update with new details` → `Specify New Details`

25. **Credentials Setup:**  
    - Configure Airtable credentials with appropriate permissions for read and write operations.  
    - No external authentication (e.g., OAuth) required for webhooks.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                                                                   |
|-------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow uses Airtable as a user database combined with cryptographic password hashing.     | Foundation for secure user authentication in serverless or lightweight backend systems.          |
| Password hashing should be done with a strong algorithm (bcrypt recommended).                     | n8n Crypto node can be configured accordingly; verify chosen algorithm and salt usage.          |
| Webhooks are exposed publicly and must be secured externally (e.g., API Gateway, IP filtering).  | n8n does not provide built-in authentication on webhooks; consider external layers for security. |
| Error handling uses "continueRegularOutput" on Airtable to avoid workflow failure on API errors. | Ensures workflow robustness when Airtable returns empty or malformed data.                        |
| Sticky Notes present are empty and can be used to add documentation or reminders inside n8n UI.  | Useful for team collaboration or future enhancement notes.                                       |

---

**Disclaimer:** The provided text is extracted exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.
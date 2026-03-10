Create WordPress users from KlickTipp and tag contacts based on comments

https://n8nworkflows.xyz/workflows/create-wordpress-users-from-klicktipp-and-tag-contacts-based-on-comments-13721


# Create WordPress users from KlickTipp and tag contacts based on comments

This document provides a technical reference for the n8n workflow: **Create WordPress users from KlickTipp and tag contacts based on comments**.

---

### 1. Workflow Overview
This workflow automates the bidirectional synchronization between **KlickTipp** (email marketing) and **WordPress** (CMS). It serves two primary business functions:

1.  **User Provisioning:** Automatically creates a WordPress `subscriber` account when a lead signs up via a KlickTipp RAW form, generates secure credentials, and saves them back to KlickTipp for automated welcome emails.
2.  **Engagement Tracking:** Periodically fetches WordPress comments, associates them with KlickTipp contacts, records the comment text in the CRM, and applies specific tags based on the landing page where the interaction occurred.

The logic is divided into the following functional blocks:
*   **Inbound Handling:** Triggering and filtering form submissions.
*   **Identity Logic:** Generating unique usernames and random passwords.
*   **WordPress Management:** Creating users or updating existing ones.
*   **KlickTipp Feedback:** Syncing WordPress IDs and credentials back to the contact.
*   **Comment Sync:** Fetching activity and cleaning HTML content for CRM storage.
*   **Segmentation:** Logical switching to apply tags based on page URLs.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Filtering
Processes incoming signups and ensures only those with explicit consent proceed.
*   **Nodes Involved:** `Form signup for Wordpress in KlickTipp form`, `Filter`.
*   **Node Details:**
    *   **KlickTipp Trigger:** Listens for RAW form submissions via a dedicated webhook.
    *   **Filter:** Checks if `field229493` equals "I give consent". This prevents unauthorized account creation.

#### 2.2 Identity & Security Logic
Prepares the data needed for a new WordPress account.
*   **Nodes Involved:** `Set username & password`.
*   **Node Details:**
    *   **Username Generation:** Uses an expression to combine First Name and Last Name, normalizes text (removing accents/special characters), converts to lowercase, and appends the KlickTipp ID for uniqueness.
    *   **Password Generation:** Uses a JavaScript math expression to generate a 12-digit numeric string.

#### 2.3 WordPress Account Management
Handles the creation or identification of users on the WordPress site.
*   **Nodes Involved:** `Generate Wordpress user from signup`, `Generate Wordpress user from signup1`, `Search WP Users with role Subscriber1`, `Update a user`.
*   **Node Details:**
    *   **Creation (Success Path):** Uses the WordPress node to create a user. On success, it calls the WordPress API via HTTP Request to explicitly set the role to `subscriber`.
    *   **Error Path (Existing User):** If the user exists, the workflow searches for that user by email and updates their nickname to include the KlickTipp ID.

#### 2.4 WordPress Comment Sync
A scheduled process to pull site engagement into the CRM.
*   **Nodes Involved:** `Pull comments trigger`, `Get last comments from Wordpress`, `Search WP Users with role Subscriber`, `Check for contact existence`.
*   **Node Details:**
    *   **Schedule Trigger:** Runs at defined intervals (default: 8:00 AM).
    *   **HTTP Request (WP API):** Fetches approved comments from the last 24 hours using the `after` query parameter with an ISO timestamp.
    *   **Contact Verification:** Searches KlickTipp by the commenter's email. If not found, the flow terminates via a `NoOp` node to avoid orphaned data.

#### 2.5 Data Merging & Segmentation
Cleans the comment data and applies marketing tags.
*   **Nodes Involved:** `Write comment into contact field`, `Check relevant segment`, `Tag contact`, `Tag contact1`.
*   **Node Details:**
    *   **Regex Cleaning:** The `Write comment` node uses `.replace(/<[^>]*>/g, '')` to strip HTML tags from the WordPress comment before saving it to KlickTipp field `field229468`.
    *   **Switch Logic:** Evaluates the WordPress comment `link`. If it contains specific URL paths (e.g., "integration" or "ABC"), it routes to specific tagging nodes.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Form signup... | KlickTipp Trigger | Entry Point (Webhook) | None | Filter | 1. Inbound Form submissions and comments |
| Filter | Filter | Consent Verification | Form signup... | Set username & password | 2. Filter requests |
| Set username... | Set | Identity Generation | Filter | Generate Wordpress... | 3. Identity & Security Logic |
| Generate Wordpress... | WordPress | User Creation | Set username... | Generate Wordpress...1, Search WP Users...1 | 4. WordPress Account Management |
| Generate Wordpress...1 | HTTP Request | Role Assignment | Generate Wordpress... | Update Wordpress... | 4. WordPress Account Management |
| Search WP Users...1 | HTTP Request | Existing User Lookup | Generate Wordpress... | Update a user | 5. Error handling |
| Update a user | WordPress | Metadata Update | Search WP Users...1 | Update Wordpress...1 | 5. Error handling |
| Update Wordpress... | KlickTipp | Sync Credentials | Generate Wordpress...1 | None | 6. Data Merging & Writing |
| Update Wordpress...1 | KlickTipp | Sync ID/User | Update a user | None | 6. Data Merging & Writing |
| Pull comments trigger | Schedule Trigger | Entry Point (Time) | None | Get last comments... | 1. Inbound Form submissions and comments |
| Get last comments... | HTTP Request | Fetch WP Comments | Pull comments trigger | Search WP Users... | 1. Inbound Form submissions and comments |
| Search WP Users... | HTTP Request | WP User Validation | Get last comments... | Check for contact existence | 4. WordPress Account Management |
| Check for contact... | KlickTipp | CRM Presence Check | Search WP Users... | Write comment..., No Op | 6. Data Merging & Writing |
| Write comment... | KlickTipp | Save cleaned text | Check for contact... | Check relevant segment | 6. Data Merging & Writing |
| Check relevant segment | Switch | Logic Routing | Write comment... | Tag contact, Tag contact1 | 7. Segmentation & Tagging |
| Tag contact | KlickTipp | Apply Tag (ID 14162905) | Check relevant segment | None | 7. Segmentation & Tagging |
| Tag contact1 | KlickTipp | Apply Tag (ID 14176702) | Check relevant segment | None | 7. Segmentation & Tagging |
| No Operation... | NoOp | Error handling | Check for contact existence | None | 5. Error handling |

---

### 4. Reproducing the Workflow from Scratch

1.  **KlickTipp Setup:** 
    *   Create custom fields in KlickTipp for `WP_User_ID`, `WP_Username`, `WP_Password`, and `Comment_Text`.
    *   Configure a RAW HTML form in KlickTipp and set its Redirect/Success URL to the n8n Webhook URL generated by the trigger node.
2.  **WordPress Setup:**
    *   Generate an **Application Password** in WordPress for an administrator account.
    *   Enable the REST API.
3.  **Path A: User Creation:**
    *   Add a **KlickTipp Trigger** node.
    *   Add a **Filter** node to check for a "consent" field value.
    *   Add a **Set** node. Use an expression to sanitize the username (normalize/replace regex) and generate a 12-digit password using `Math.random()`.
    *   Add a **WordPress** node (Action: Create User). Enable "Continue on Fail" to handle existing users.
    *   Connect the success output to an **HTTP Request** node calling `POST /wp-json/wp/v2/users/{id}` to set the `roles` to `["subscriber"]`.
    *   Connect the failure output to a search/update sequence to link existing users.
4.  **Path B: Comment Sync:**
    *   Add a **Schedule Trigger** (e.g., daily).
    *   Add an **HTTP Request** node to WordPress: `GET /wp-json/wp/v2/comments`. Filter by `status=approve` and `after={{ $now.minus({hours: 24}) }}`.
    *   Add a **KlickTipp** node (Action: Get Subscriber) to ensure the commenter exists in your CRM.
    *   Add a **KlickTipp** node (Action: Update Subscriber). Use regex in the expression for the comment field to strip `<p>` and other tags.
    *   Add a **Switch** node evaluating the `link` property. Create routes for specific Landing Page URLs.
    *   Add **KlickTipp** nodes (Action: Tag Contact) for each switch output.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Community Node Disclaimer | This workflow utilizes KlickTipp community nodes for n8n. |
| WordPress REST API Documentation | Used for user role updates and comment fetching. [WP API Reference](https://developer.wordpress.org/rest-api/) |
| Regex for HTML Stripping | `replace(/<[^>]*>/g, '')` ensures clean text in CRM fields. |
| Username Normalization | Handles UTF-8 characters (accents) to ensure WP compatibility. |
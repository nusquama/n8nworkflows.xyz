Automate SEO Title & Description Updates for WordPress with Yoast SEO API

https://n8nworkflows.xyz/workflows/automate-seo-title---description-updates-for-wordpress-with-yoast-seo-api-7180


# Automate SEO Title & Description Updates for WordPress with Yoast SEO API

### 1. Workflow Overview

This workflow automates the update of SEO metadata—specifically the SEO Title and Meta Description—on a WordPress post using the Yoast SEO API. It is designed primarily for users managing WordPress sites with the Yoast SEO plugin and a custom API endpoint installed. The workflow consists of three logical blocks:

- **1.1 Manual Trigger & Settings Initialization:** Receives a manual trigger to start the process and sets configuration parameters such as the WordPress site URL.
- **1.2 SEO Metadata Update via HTTP Request:** Sends a POST request to the custom Yoast SEO API endpoint to update the SEO Title and Meta Description for a specified post.
- **1.3 Documentation & Configuration Notes:** Provides essential information about prerequisites and setup conditions to ensure proper workflow operation.

---

### 2. Block-by-Block Analysis

#### 2.1 Manual Trigger & Settings Initialization

- **Overview:**  
  This block initiates the workflow execution manually and establishes necessary configuration variables, such as the WordPress site URL.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Settings (Set Node)

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - *Type & Role:* Manual Trigger node, initiates workflow execution on user demand.  
    - *Configuration:* No parameters; user manually triggers the workflow.  
    - *Expressions:* None.  
    - *Input/Output:* No input; outputs trigger to the "Settings" node.  
    - *Version:* 1 (basic manual trigger functionality).  
    - *Edge Cases:* None significant; manual trigger requires user interaction.  
    - *Sub-Workflow:* None.

  - **Settings**  
    - *Type & Role:* Set node, defines workflow configuration variables.  
    - *Configuration:* Assigns a string variable `wordpress URL` with value `"https://example.com/"` (placeholder URL to be replaced with actual site).  
    - *Expressions:* Static string assignment, no dynamic expressions.  
    - *Input/Output:* Inputs trigger from manual node; outputs to HTTP request node.  
    - *Version:* 3.4.  
    - *Edge Cases:* If URL is incorrect or missing trailing slash, subsequent HTTP requests may fail. User must configure correctly.  
    - *Sub-Workflow:* None.

---

#### 2.2 SEO Metadata Update via HTTP Request

- **Overview:**  
  This block performs the core action by sending a POST request to the Yoast SEO API endpoint on the WordPress site to update the SEO Title and Meta Description of a specified post.

- **Nodes Involved:**  
  - HTTP Request - Update Yoast Meta

- **Node Details:**

  - **HTTP Request - Update Yoast Meta**  
    - *Type & Role:* HTTP Request node, performs authenticated POST request to update SEO metadata.  
    - *Configuration:*  
      - URL: Constructed dynamically using the variable `wordpress URL` from the Settings node appended with `wp-json/yoast-api/v1/update-meta`.  
      - Method: POST  
      - Body: Sends JSON parameters including `post_id` (hardcoded as `519`), `yoast_title` ("Demo SEO Title"), and `yoast_description` ("Demo SEO Description").  
      - Authentication: Uses predefined credentials of type `wordpressApi` (OAuth2 or API key depending on setup).  
      - Retry on failure: Enabled to handle transient network or server issues.  
    - *Expressions:* Uses expression `={{ $json["wordpress URL"] }}wp-json/yoast-api/v1/update-meta` to construct endpoint URL dynamically.  
    - *Input/Output:* Input from Settings node; output is response from WordPress API.  
    - *Version:* 4.2  
    - *Edge Cases:*  
      - Authentication failure if credentials are invalid or expired.  
      - HTTP errors (404 if endpoint not found, 500 if plugin malfunctioning).  
      - Incorrect `post_id` or malformed body causing update failure.  
      - Network issues causing timeouts or retries.  
    - *Sub-Workflow:* None.

---

#### 2.3 Documentation & Configuration Notes

- **Overview:**  
  Provides users with important notes about the workflow’s purpose, prerequisites, and configuration parameters necessary for successful execution.

- **Nodes Involved:**  
  - Sticky Note

- **Node Details:**

  - **Sticky Note**  
    - *Type & Role:* Informational node to document workflow context and usage instructions.  
    - *Configuration:* Contains markdown-formatted text explaining the workflow’s function, prerequisites (custom plugin installed on WordPress), and configuration details (manual setting of `post_id` and WordPress site URL).  
    - *Input/Output:* No input or output connections; purely informational.  
    - *Version:* 1  
    - *Edge Cases:* None.  
    - *Sub-Workflow:* None.

---

### 3. Summary Table

| Node Name                      | Node Type         | Functional Role                       | Input Node(s)          | Output Node(s)                   | Sticky Note                                                                                                                      |
|--------------------------------|-------------------|------------------------------------|------------------------|---------------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’  | Manual Trigger    | Initiates workflow on user command | None                   | Settings                        |                                                                                                                                  |
| Settings                      | Set Node          | Sets configuration variable (WordPress URL) | When clicking ‘Test workflow’ | HTTP Request - Update Yoast Meta |                                                                                                                                  |
| HTTP Request - Update Yoast Meta | HTTP Request     | Sends POST request to update SEO metadata in WordPress | Settings                | None                            |                                                                                                                                  |
| Sticky Note                   | Sticky Note       | Provides documentation and prerequisite notes | None                   | None                            | ## Yoast SEO Update  This workflow updates the **SEO Title** and **Meta Description** of a post via a request to a custom API endpoint.  **Prerequisites:** The provided custom plugin must be installed and active on the WordPress site.  **Configuration:** The `post_id` of the post to be modified is set manually in the **HTTP Request - Update Yoast Meta** node. The WordPress site URL is set in the **Settings** node.|

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Node Type: Manual Trigger  
   - Name: `When clicking ‘Test workflow’`  
   - No parameters needed.

2. **Create Set Node**  
   - Node Type: Set  
   - Name: `Settings`  
   - Add an assignment:  
     - Variable name: `wordpress URL`  
     - Type: String  
     - Value: `https://example.com/` (replace with your actual WordPress site URL, ensure trailing slash included)  
   - Connect output of `When clicking ‘Test workflow’` to this node.

3. **Create HTTP Request Node**  
   - Node Type: HTTP Request  
   - Name: `HTTP Request - Update Yoast Meta`  
   - HTTP Method: POST  
   - URL: Use an expression to concatenate the `wordpress URL` variable and the API path:  
     ```
     ={{ $json["wordpress URL"] }}wp-json/yoast-api/v1/update-meta
     ```  
   - Authentication: Select or create credentials for your WordPress site (`wordpressApi` type, typically OAuth2 or API key depending on your setup).  
   - Body Parameters (send as JSON):  
     - `post_id`: `519` (replace with your target post ID)  
     - `yoast_title`: `Demo SEO Title` (replace as needed)  
     - `yoast_description`: `Demo SEO Description` (replace as needed)  
   - Enable "Retry On Fail" to automatically retry on transient errors.  
   - Connect output of `Settings` node to this node.

4. **Add Sticky Note for Documentation** (optional but recommended)  
   - Create Sticky Note node with content:  
     ```
     ## Yoast SEO Update

     This workflow updates the **SEO Title** and **Meta Description** of a post via a request to a custom API endpoint.

     **Prerequisites:**  
     * The provided custom plugin must be installed and active on the WordPress site.

     **Configuration:**  
     * The `post_id` of the post to be modified is set manually in the **HTTP Request - Update Yoast Meta** node.  
     * The WordPress site URL is set in the **Settings** node.
     ```  
   - Position as desired; no connections needed.

5. **Credentials Setup**  
   - Ensure you have valid WordPress API credentials configured in n8n under the name used in the HTTP Request node (`wordpressApi`). This typically involves OAuth2 or API key/token authentication with your WordPress site.

6. **Testing**  
   - Manually trigger the workflow to test the update.  
   - Monitor node execution for errors, especially in the HTTP Request node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                  | Context or Link                                 |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------|
| This workflow requires a custom WordPress plugin that exposes the Yoast SEO update endpoint at `/wp-json/yoast-api/v1/update-meta`.                            | WordPress site with custom Yoast SEO API plugin |
| The `post_id` is hardcoded in the HTTP Request node and must exist on the WordPress site. Adjust as needed.                                                  | Workflow configuration                            |
| WordPress URL must include trailing slash to correctly form the API endpoint URL.                                                                             | Workflow configuration                            |

---

**Disclaimer:** The content provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data manipulated are legal and public.
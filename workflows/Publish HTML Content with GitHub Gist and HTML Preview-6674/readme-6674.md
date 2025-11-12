Publish HTML Content with GitHub Gist and HTML Preview

https://n8nworkflows.xyz/workflows/publish-html-content-with-github-gist-and-html-preview-6674


# Publish HTML Content with GitHub Gist and HTML Preview

### 1. Workflow Overview

This workflow automates the process of publishing generated HTML content as a private GitHub Gist and provides a publicly accessible URL to preview the HTML content via the HTML Preview service. It is designed to be triggered by another workflow that supplies HTML content as input and returns a shareable URL for the rendered HTML. The workflow is structured into the following logical blocks:

- **1.1 Input Reception:** Receives HTML content from an external workflow invocation.
- **1.2 GitHub Gist Creation:** Publishes the HTML content as a private GitHub Gist using the GitHub API.
- **1.3 URL Generation:** Constructs a preview URL that allows viewing the HTML content via htmlpreview.github.io.
- **1.4 Documentation/Instruction:** Provides guidance on configuring authentication and usage through a sticky note.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block serves as the entry point, configured to be triggered by another workflow, passing HTML content into this workflow for processing.

- **Nodes Involved:**  
  - `When Executed by Another Workflow`

- **Node Details:**

  - **Node Name:** When Executed by Another Workflow  
    - **Type and Role:** `executeWorkflowTrigger` node; triggers this workflow and accepts input parameters.  
    - **Configuration:**  
      - Accepts a single input parameter named `html`.  
      - Version 1.1 of the node type.  
    - **Key Expressions/Variables:** None internally; input parameter is `$json.html`.  
    - **Input/Output Connections:**  
      - No input connections (trigger node).  
      - Outputs to `Create Gist` node.  
    - **Potential Failures:**  
      - If no `html` parameter is provided, downstream nodes may fail due to missing content.  
      - Triggering workflow must provide valid HTML content.  
    - **Sub-workflow Reference:** This node is designed to be called by other workflows, expecting an input named `html`.

#### 2.2 GitHub Gist Creation

- **Overview:**  
  This block sends an HTTP POST request to the GitHub API to create a private gist containing the HTML content received. It uses authentication via a GitHub API key.

- **Nodes Involved:**  
  - `Create Gist`

- **Node Details:**

  - **Node Name:** Create Gist  
    - **Type and Role:** `httpRequest` node; performs a POST request to GitHub's Gist API to create a new gist.  
    - **Configuration:**  
      - URL: `https://api.github.com/gists`  
      - Method: POST  
      - Body: JSON with fields:  
        - `description`: "Auto-rendered HTML"  
        - `public`: false (private gist)  
        - `files`: object containing one file named `report.html` whose content is the JSON stringified HTML input (`{{ JSON.stringify($json.html) }}`).  
      - Authentication: Generic HTTP Header Authentication using a stored credential named `GitHub API`.  
      - Headers: Includes `User-Agent: n8n` as required by GitHub API.  
      - Retry on Fail: Enabled with 5000 ms wait between attempts to handle transient errors.  
    - **Key Expressions/Variables:**  
      - Body content dynamically includes the HTML input via `{{ JSON.stringify($json.html) }}`.  
    - **Input/Output Connections:**  
      - Input from `When Executed by Another Workflow` node.  
      - Output to `Set URL` node.  
    - **Version-specific Requirements:** Uses version 4.2 of the node type for enhanced HTTP request features.  
    - **Potential Failures:**  
      - Authentication failure if GitHub API key is invalid or missing.  
      - API rate limiting or network errors.  
      - Malformed HTML input leading to invalid JSON serialization.  
      - API changes in GitHub Gist endpoint.  
    - **Sticky Note Reference:** Requires API key configuration as noted in the sticky note.

#### 2.3 URL Generation

- **Overview:**  
  This block constructs a publicly accessible URL that uses the `htmlpreview.github.io` service to render the raw gist content, enabling HTML preview in browsers.

- **Nodes Involved:**  
  - `Set URL`

- **Node Details:**

  - **Node Name:** Set URL  
    - **Type and Role:** `set` node; creates a new JSON field `URL` containing the preview link.  
    - **Configuration:**  
      - Assigns a new string field `URL` with the value constructed by concatenating `https://htmlpreview.github.io/?` with the raw URL of the gist file: `{{$json.files["report.html"].raw_url}}`.  
    - **Key Expressions/Variables:**  
      - Uses expression: `={{ "https://htmlpreview.github.io/?" + $json.files["report.html"].raw_url }}`  
    - **Input/Output Connections:**  
      - Input from `Create Gist` node.  
      - No further output connections (end of workflow).  
    - **Version-specific Requirements:** Uses version 3.4 of the node type.  
    - **Potential Failures:**  
      - If the `Create Gist` node fails or returns unexpected response structure, `$json.files["report.html"].raw_url` may be undefined.  
      - URL construction may break if API changes response format.

#### 2.4 Documentation/Instruction

- **Overview:**  
  Provides user instructions on setting up the GitHub API key and using the workflow.

- **Nodes Involved:**  
  - `Sticky Note`

- **Node Details:**

  - **Node Name:** Sticky Note  
    - **Type and Role:** `stickyNote` node; non-executing documentation aid inside the workflow editor.  
    - **Configuration:**  
      - Content explains:  
        - Add GitHub API key to the `Create Gist` node authentication header.  
        - The workflow input is an HTML report.  
        - The output is a URL rendering the HTML.  
    - **Input/Output Connections:** None (standalone).  
    - **Potential Failures:** None (documentation only).  

---

### 3. Summary Table

| Node Name                    | Node Type               | Functional Role                     | Input Node(s)                  | Output Node(s)             | Sticky Note                                                                                         |
|------------------------------|------------------------|-----------------------------------|-------------------------------|----------------------------|--------------------------------------------------------------------------------------------------|
| When Executed by Another Workflow | executeWorkflowTrigger | Entry trigger, receives HTML input | None                          | Create Gist                |                                                                                                  |
| Create Gist                  | httpRequest            | Creates private GitHub Gist with HTML content | When Executed by Another Workflow | Set URL                    | Add GitHub API key to "Create Gist" node. First, add your API key to the auth header in the create gist node. Then, you can pass HTML reports into this workflow as the input and receive a URL where the HTML is displayed as the output. |
| Set URL                     | set                    | Generates preview URL using gist raw URL | Create Gist                   | None                       |                                                                                                  |
| Sticky Note                 | stickyNote             | Documentation with setup instructions | None                          | None                       | ## Add GitHub API key to "Create Gist" node First, add your API key to the auth header in the create gist node. Then, you can pass HTML reports into this workflow as the input and receive a URL where the HTML is displayed as the output. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node**  
   - Add a new node of type **Execute Workflow Trigger** (`When Executed by Another Workflow`).  
   - Configure it to accept a workflow input parameter:  
     - Name: `html`  
     - Type: (default) - accepts string or JSON content.  
   - This node will act as the entry point expecting HTML content.

2. **Create the GitHub Gist HTTP Request Node**  
   - Add a new **HTTP Request** node named `Create Gist`.  
   - Set the HTTP Method to `POST`.  
   - Set the URL to `https://api.github.com/gists`.  
   - Under the **Authentication** section, select **HTTP Header Auth** and configure credentials with your GitHub API token:  
     - Header name: `Authorization`  
     - Header value: `token YOUR_GITHUB_PERSONAL_ACCESS_TOKEN`  
   - Add an additional header:  
     - Name: `User-Agent`  
     - Value: `n8n` (required by GitHub API)  
   - In the Body Parameters, choose to send a JSON body with the following structure:  
     ```json
     {
       "description": "Auto-rendered HTML",
       "public": false,
       "files": {
         "report.html": {
           "content": {{ JSON.stringify($json.html) }}
         }
       }
     }
     ```  
   - Enable **Send Body** and **Send Headers**.  
   - Enable **Retry on Fail** with a wait time of 5000 ms.

3. **Create the URL Setter Node**  
   - Add a **Set** node named `Set URL`.  
   - Add a new field:  
     - Name: `URL`  
     - Type: String  
     - Value expression: `={{ "https://htmlpreview.github.io/?" + $json.files["report.html"].raw_url }}`  
   - This constructs the public URL to preview the gist content.

4. **Connect the Nodes**  
   - Connect `When Executed by Another Workflow` node output to `Create Gist` node input.  
   - Connect `Create Gist` node output to `Set URL` node input.

5. **Add Documentation (Optional)**  
   - Add a **Sticky Note** node with the following content:  
     ```
     ## Add GitHub API key to "Create Gist" node
     First, add your API key to the auth header in the create gist node.

     Then, you can pass HTML reports into this workflow as the input and receive a URL where the HTML is displayed as the output.
     ```

6. **Finalize and Save**  
   - Save the workflow.  
   - Ensure your GitHub API credentials are securely stored in n8n credentials manager and referenced properly.  
   - Test the workflow by triggering it from another workflow with an `html` parameter containing valid HTML content.

---

### 5. General Notes & Resources

| Note Content                                                                                       | Context or Link                                  |
|--------------------------------------------------------------------------------------------------|-------------------------------------------------|
| Ensure your GitHub personal access token has `gist` scope enabled to allow gist creation.        | GitHub API documentation: https://docs.github.com/en/rest/gists/gists#create-a-gist |
| The preview URL uses `https://htmlpreview.github.io/` which is a free service to render HTML files from raw URLs. | HTML Preview Service: https://htmlpreview.github.io/ |
| The workflow requires n8n version that supports HTTP Request node with Generic HTTP Header Auth (recommended v0.180.0 or later). | n8n Release Notes: https://docs.n8n.io/reference/changelog/ |
| This workflow is designed to be used as a sub-workflow for easier integration in larger automation processes. | n8n Sub-Workflows: https://docs.n8n.io/workflows/execute-workflow-trigger/ |

---

**Disclaimer:**  
The provided workflow and documentation are based exclusively on an automated n8n workflow. All processed data is legal and public. The workflow respects content policies and contains no illegal or offensive material.
WebhookDocs: generate swagger preview of your active workflows

https://n8nworkflows.xyz/workflows/webhookdocs--generate-swagger-preview-of-your-active-workflows-4270


# WebhookDocs: generate swagger preview of your active workflows

### 1. Workflow Overview

This workflow named **WebhookDocs: generate swagger preview of your active workflows** is designed to dynamically generate a Swagger (OpenAPI 2.0) specification preview of all active workflows containing webhooks in an n8n instance. Its primary use case is to provide API documentation for the active webhook endpoints of the user’s workflows, enabling easier understanding, testing, and integration.

The workflow logically divides into the following blocks:

- **1.1 Input Reception:** Accept an incoming HTTP request at a specific webhook endpoint to trigger the Swagger generation process.
- **1.2 Active Workflow Retrieval:** Query the n8n instance via API to obtain all active workflows and their node details.
- **1.3 Swagger Specification Generation:** Analyze retrieved workflows to extract webhook nodes, their paths, HTTP methods, and response configurations to build a Swagger YAML specification.
- **1.4 Swagger UI Response:** Return a dynamic HTML page embedding Swagger UI to render the generated Swagger YAML for user-friendly API preview.
- **1.5 User Guidance Note:** A sticky note node providing instructions on how to add parameter metadata comments inside webhook node notes for enhanced Swagger documentation.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block sets up the initial webhook that users call to trigger the whole Swagger spec generation.

- **Nodes Involved:**  
  - Get Swagger

- **Node Details:**  
  - **Get Swagger**  
    - Type: Webhook (HTTP Trigger)  
    - Configuration:  
      - Path: `/swagger`  
      - HTTP Methods: Defaults to GET (implied)  
      - Response Mode: "responseNode" (enables downstream node to send response)  
    - Inputs: None (entry point)  
    - Outputs: Connected to the "n8n" node to fetch active workflows  
    - Edge Cases:  
      - If the webhook path conflicts with existing endpoints, the request may fail or hit unintended flows.  
      - No authentication configured; anyone knowing the path can trigger the workflow.

#### 2.2 Active Workflow Retrieval

- **Overview:**  
  This block calls the n8n internal API to fetch all active workflows and their detailed node structures needed for Swagger generation.

- **Nodes Involved:**  
  - n8n

- **Node Details:**  
  - **n8n**  
    - Type: n8n API Node (internal API integration)  
    - Configuration:  
      - Filter: Only "activeWorkflows" set to true, meaning only workflows currently enabled will be fetched  
      - Credentials: Uses "n8nApi" credential (configured with API access to the n8n instance)  
    - Inputs: From Get Swagger node  
    - Outputs: To Code node  
    - Edge Cases:  
      - API errors if credentials expire or are revoked  
      - Large number of active workflows may cause timeout or performance issues  
      - Network issues between n8n nodes and API

#### 2.3 Swagger Specification Generation

- **Overview:**  
  This block processes the fetched workflows to identify webhook nodes, their HTTP methods, paths, and response nodes, then constructs a Swagger 2.0 YAML document string representing all active webhook endpoints.

- **Nodes Involved:**  
  - Code

- **Node Details:**  
  - **Code**  
    - Type: Code (JavaScript)  
    - Configuration:  
      - Custom JS code scans all active workflows’ nodes, extracting those of type `webhook` and linked `respondToWebhook` nodes.  
      - Uses helper functions to:  
        - Sanitize strings for Swagger compatibility (`safe` function)  
        - Traverse workflow connections to find valid response targets linked to webhooks  
      - Builds Swagger YAML string with:  
        - API metadata (title, version, host from request headers fallback)  
        - Paths section dynamically generated from webhook nodes  
        - Parameters extracted from webhook notes using `//@query` and `//@body` annotations  
        - Response content types inferred from response node configuration  
      - Outputs: JSON with two properties:  
        - `text`: The Swagger YAML string  
        - `nodes`: Internal structure of webhook nodes for debugging or further use  
    - Inputs: From n8n node (active workflows data)  
    - Outputs: To Respond to Webhook node  
    - Edge Cases:  
      - Workflow nodes lacking proper webhook or response node types will be skipped  
      - Errors in parsing notes or malformed annotations may cause incomplete parameter documentation  
      - If the `host` header is missing in the original request, a default host is used ('n8n.instance.com'), which may be inaccurate  
      - Large or complex workflows may slow script execution or hit JS runtime limits

#### 2.4 Swagger UI Response

- **Overview:**  
  This block sends back an HTML page embedding Swagger UI that renders the generated Swagger YAML specification, providing a live API documentation preview in the browser.

- **Nodes Involved:**  
  - Respond to Webhook

- **Node Details:**  
  - **Respond to Webhook**  
    - Type: Respond to Webhook (HTTP response node)  
    - Configuration:  
      - Respond With: Text (HTML)  
      - Response Body: Custom HTML embedding Swagger UI libraries from CDN, loading the Swagger YAML text dynamically via JS variable interpolation  
      - Dynamic content: `{{ $json.text }}` injects Swagger YAML from previous node’s output  
    - Inputs: From Code node  
    - Outputs: None (terminates workflow with HTTP response)  
    - Edge Cases:  
      - If `$json.text` is undefined or empty, Swagger UI will fail to load properly  
      - Network restrictions blocking CDN resources will prevent UI from rendering  
      - Very large Swagger specs may slow page load or cause browser memory issues

#### 2.5 User Guidance Note

- **Overview:**  
  This sticky note provides the user with instructions on how to add parameter metadata annotations inside webhook node notes to improve Swagger parameter documentation quality.

- **Nodes Involved:**  
  - Sticky Note

- **Node Details:**  
  - **Sticky Note**  
    - Type: Sticky Note (documentation node)  
    - Content:  
      ```
      ## Configure webhooks

      In order to support parameter labels you have to open the note sections of every webhook and add the following text

      //@body field_name string description
      //@query field_name string description
      ```
    - Position: Informational, no inputs or outputs  
    - Edge Cases: None (purely instructional)

---

### 3. Summary Table

| Node Name        | Node Type                    | Functional Role                     | Input Node(s)     | Output Node(s)       | Sticky Note                                                                                         |
|------------------|------------------------------|-----------------------------------|-------------------|----------------------|---------------------------------------------------------------------------------------------------|
| Get Swagger      | Webhook                      | Entry point webhook to trigger Swagger generation | None              | n8n                  |                                                                                                   |
| n8n              | n8n API                      | Fetch active workflows from n8n instance | Get Swagger       | Code                 |                                                                                                   |
| Code             | Code (JavaScript)            | Generate Swagger YAML from workflows data | n8n               | Respond to Webhook    |                                                                                                   |
| Respond to Webhook| Respond to Webhook           | Return Swagger UI HTML page with rendered Swagger spec | Code              | None                 |                                                                                                   |
| Sticky Note      | Sticky Note                  | User instructions about webhook parameter annotations | None              | None                 | ## Configure webhooks; In order to support parameter labels you have to open the note sections of every webhook and add the following text //@body field_name string description //@query field_name string description |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook node**  
   - Name: `Get Swagger`  
   - Type: Webhook  
   - Set Path to `swagger`  
   - Leave HTTP Method default (GET)  
   - Set Response Mode to `responseNode`  
   - No authentication configured (optional)  

2. **Create an n8n API node**  
   - Name: `n8n`  
   - Type: n8n API node (under Credentials)  
   - Set filters: `activeWorkflows` = `true` to fetch only active workflows  
   - Select or create credential: `n8nApi` (OAuth2 or API key with rights to read workflows)  
   - Connect input from `Get Swagger` node  

3. **Create a Code node**  
   - Name: `Code`  
   - Type: Code (JavaScript)  
   - Paste the JavaScript code to:  
     - Iterate over all workflows received  
     - Identify `webhook` and linked `respondToWebhook` nodes  
     - Parse notes for parameter annotations (`//@query` and `//@body`)  
     - Build a Swagger 2.0 YAML string with paths, methods, parameters, and responses  
     - Include dynamic host detection from request headers or use default  
     - Return JSON object with `text` (Swagger YAML) and `nodes` (debug info)  
   - Connect input from `n8n` node  

4. **Create a Respond to Webhook node**  
   - Name: `Respond to Webhook`  
   - Type: Respond to Webhook  
   - Set `Respond With` to `Text`  
   - Set Response Body to HTML embedding Swagger UI from CDN, with embedded JS to parse and render the Swagger YAML from previous node’s JSON `text` property:  
     ```html
     <!DOCTYPE html>
     <html>
     <head>
       <title>Swagger UI from YAML Text</title>
       <link rel="stylesheet" href="https://unpkg.com/swagger-ui-dist/swagger-ui.css">
     </head>
     <body>
       <div id="swagger-ui"></div>

       <script src="https://unpkg.com/swagger-ui-dist/swagger-ui-bundle.js"></script>
       <script src="https://unpkg.com/js-yaml@4.1.0/dist/js-yaml.min.js"></script>
       <script>
         const yamlText = `{{ $json.text }}`;
         const spec = jsyaml.load(yamlText);
         const ui = SwaggerUIBundle({
           spec: spec,
           dom_id: "#swagger-ui"
         });
       </script>
     </body>
     </html>
     ```  
   - Connect input from `Code` node  

5. **Add a Sticky Note node**  
   - Name: `Sticky Note`  
   - Content:  
     ```
     ## Configure webhooks

     In order to support parameter labels you have to open the note sections of every webhook and add the following text

     //@body field_name string description
     //@query field_name string description
     ```  
   - Position it clearly in the workspace for user reference  

6. **Connect nodes in order:**  
   - `Get Swagger` → `n8n` → `Code` → `Respond to Webhook`  

7. **Credential Setup:**  
   - Create or verify `n8nApi` credential with permissions to read workflows  
   - Assign credential to `n8n` node  

8. **Save and activate the workflow**  

---

### 5. General Notes & Resources

| Note Content                                                                                                               | Context or Link                                                                                  |
|----------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| To enable detailed parameter documentation in generated Swagger, add note annotations `//@body` and `//@query` inside each webhook node’s note field with name, type, and description. | Sticky Note content in workflow                                                                |
| Swagger UI resources are loaded from CDN URLs:  
- https://unpkg.com/swagger-ui-dist/swagger-ui.css  
- https://unpkg.com/swagger-ui-dist/swagger-ui-bundle.js  
- https://unpkg.com/js-yaml@4.1.0/dist/js-yaml.min.js | Embedded in Respond to Webhook node’s HTML response body                                        |
| The generated Swagger spec uses OpenAPI 2.0 (Swagger 2.0) specification format.                                             | Standard Swagger/OpenAPI documentation format                                                  |
| Default host fallback is `n8n.instance.com` if the original incoming request does not supply a host header.                | Default in Code node                                                                              |
| This workflow requires n8n version supporting:  
- Webhook node with responseMode "responseNode" (n8n v0.152+)  
- n8n API node integration  
- Respond to Webhook node capable of sending HTML responses | Version-specific requirements inferred from node usage                                         |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.
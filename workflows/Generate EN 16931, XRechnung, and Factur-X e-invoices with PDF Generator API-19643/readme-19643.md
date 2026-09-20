Generate EN 16931, XRechnung, and Factur-X e-invoices with PDF Generator API

https://n8nworkflows.xyz/workflows/generate-en-16931--xrechnung--and-factur-x-e-invoices-with-pdf-generator-api-19643


# Generate EN 16931, XRechnung, and Factur-X e-invoices with PDF Generator API

### 1. Workflow Overview

This workflow automates the generation of standards-compliant e-invoices (EN 16931, XRechnung, and Factur-X) from structured invoice JSON payloads using the PDF Generator API. It accepts invoice data via two distinct entry points (manual execution with a built-in sample or an external POST webhook), normalizes the data, calculates financial subtotals and totals, maps the payload into Peppol BIS Billing 3.0 UBL format, routes the request to the appropriate e-invoice generation endpoint, parses the returned base64-encoded document, and converts it into a binary file ready for storage or delivery.

The logic is divided into the following functional blocks:
- **1.1 Input Reception & Configuration:** Manages workflow execution triggers (manual sample or webhook) and injects default global parameters (template ID, default output format, and profile).
- **1.2 Schema Reference (Optional):** Provides a direct optional connection to retrieve the latest PDF Generator API e-invoice schema.
- **1.3 Payload Transformation:** Processes raw invoice line items, computes item extensions, tax subtotals, and monetary totals, and maps them to a UBL-compliant structure.
- **1.4 Format Routing:** Evaluates the target output format and branches the execution path.
- **1.5 Generation & Parsing:** Calls the corresponding PDF Generator API endpoint (Factur-X, XRechnung, or EN 16931), processes the response envelope, and extracts metadata.
- **1.6 File Conversion & Handoff:** Decodes the base64 payload into a binary file with appropriate naming and MIME type conventions, forwarding it to a placeholder node for integration.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Configuration
- **Overview:** This block establishes the entry points for the workflow—either a manual trigger with a sample dataset or a production webhook endpoint—and enriches the incoming data with default generation parameters.
- **Nodes Involved:** 
  - `Manual Trigger Workflow` (`n8n-nodes-base.manualTrigger`)
  - `Webhook Trigger for Invoice` (`n8n-nodes-base.webhook`)
  - `Prepare Sample Invoice` (`n8n-nodes-base.set`)
  - `Prepare Webhook Invoice` (`n8n-nodes-base.set`)
  - `Set Invoice Parameters` (`n8n-nodes-base.set`)

- **Node Details:**
  - **Manual Trigger Workflow**
    - *Type and Technical Role:* Trigger node that starts the workflow manually.
    - *Configuration:* Default execution configuration.
    - *Input/Output:* No inputs; outputs to `Prepare Sample Invoice` and `Fetch E-Invoice Schema`.
    - *Edge Cases:* Manual runs are limited to test environments; unexpected payload mismatches must be caught downstream.
  - **Webhook Trigger for Invoice**
    - *Type and Technical Role:* Webhook trigger node listening for external HTTP POST requests.
    - *Configuration:* Path configured to `e-invoice`, method set to `POST`, response mode set to `lastNode`.
    - *Input/Output:* No inputs; outputs to `Prepare Webhook Invoice`.
    - *Edge Cases:* Unauthorized requests or malformed JSON payloads will fail parsing.
  - **Prepare Sample Invoice**
    - *Type and Technical Role:* Set node using raw mode to output a complete, static sample invoice JSON structure.
    - *Configuration:* Hardcoded JSON object containing full seller, buyer, currency, and line item arrays conforming to the requirements.
    - *Input/Output:* Connected from `Manual Trigger Workflow`; outputs to `Set Invoice Parameters`.
  - **Prepare Webhook Invoice**
    - *Type and Technical Role:* Set node using raw mode to extract the JSON body from the incoming webhook request.
    - *Configuration:* Expression: `={{ JSON.stringify($json.body) }}`
    - *Input/Output:* Connected from `Webhook Trigger for Invoice`; outputs to `Set Invoice Parameters`.
  - **Set Invoice Parameters**
    - *Type and Technical Role:* Set node acting as a configuration layer for global settings.
    - *Configuration:* Assigns `template_id` (number: `1656704`), `default_format` (string: `"facturx"`), and `facturx_profile` (string: `"en16931"`), while including other fields (`includeOtherFields: true`).
    - *Input/Output:* Connected from `Prepare Sample Invoice` and `Prepare Webhook Invoice`; outputs to `Map to UBL Invoice Format`.

---

#### 2.2 Schema Reference
- **Overview:** An independent reference node that queries the PDF Generator API schema endpoint to assist developers with payload mapping.
- **Nodes Involved:**
  - `Fetch E-Invoice Schema` (`n8n-nodes-base.httpRequest`)

- **Node Details:**
  - **Fetch E-Invoice Schema**
    - *Type and Technical Role:* HTTP Request node configured to fetch the latest schema documentation.
    - *Configuration:* GET request to `https://us1.pdfgeneratorapi.com/api/v4/einvoice/schema`. Uses predefined credential type `pdfGeneratorApi`. Note: This node is disabled by default (`disabled: true`).
    - *Input/Output:* Connected from `Manual Trigger Workflow`; has no downstream connections.
    - *Credentials:* `pdfGeneratorApi`

---

#### 2.3 Payload Transformation
- **Overview:** Transforms the normalized flat invoice object into a Peppol BIS Billing 3.0 UBL payload, computing all tax calculations, line extensions, and monetary totals programmatically via JavaScript.
- **Nodes Involved:**
  - `Map to UBL Invoice Format` (`n8n-nodes-base.code`)

- **Node Details:**
  - **Map to UBL Invoice Format**
    - *Type and Technical Role:* Code node executing custom JavaScript to build a complex UBL JSON structure.
    - *Configuration:* Custom JavaScript block iterating over input items, calculating line totals, grouping taxes by rate, generating subtotals, and mapping supplier, customer, payment means, and legal monetary totals.
    - *Key Expressions/Variables:* Reads configuration variables via `$('Set Invoice Parameters').first().json` and evaluates line quantities, unit prices, and tax percentages.
    - *Input/Output:* Connected from `Set Invoice Parameters`; outputs to `Route by Invoice Format`.
    - *Edge Cases:* Missing required fields (such as `vat_id` or `iban`) may cause validation errors downstream depending on the target format profile (e.g., XRechnung strict rules).

---

#### 2.4 Format Routing
- **Overview:** Evaluates the target output format property and directs the payload down the appropriate execution branch.
- **Nodes Involved:**
  - `Route by Invoice Format` (`n8n-nodes-base.switch`)

- **Node Details:**
  - **Route by Invoice Format**
    - *Type and Technical Role:* Switch router node.
    - *Configuration:* Evaluates `={{ $json.format }}` against three strict string matching rules: `"facturx"`, `"xrechnung"`, and `"en16931"`, mapping them to named outputs (`Factur-X`, `XRechnung`, and `EN 16931`).
    - *Input/Output:* Connected from `Map to UBL Invoice Format`; outputs to `Post to Factur-X Endpoint`, `Post to XRechnung Endpoint`, and `Post to EN 16931 Endpoint` respectively.

---

#### 2.5 Generation & Parsing
- **Overview:** Dispatches the UBL payload to the appropriate PDF Generator API endpoint and processes the uniform response envelope.
- **Nodes Involved:**
  - `Post to Factur-X Endpoint` (`n8n-nodes-base.httpRequest`)
  - `Post to XRechnung Endpoint` (`n8n-nodes-base.httpRequest`)
  - `Post to EN 16931 Endpoint` (`n8n-nodes-base.httpRequest`)
  - `Parse API Response` (`n8n-nodes-base.code`)

- **Node Details:**
  - **Post to Factur-X Endpoint**
    - *Type and Technical Role:* HTTP Request node targeting the Factur-X generation service.
    - *Configuration:* POST request to `https://us1.pdfgeneratorapi.com/api/v4/einvoice/facturx`. JSON body wraps template settings (`template: { id, data }`), profile, output type (`base64`), and document name. Uses predefined credential type `pdfGeneratorApi`.
    - *Input/Output:* Connected from branch 0 of `Route by Invoice Format`; outputs to `Parse API Response`.
    - *Credentials:* `pdfGeneratorApi`
  - **Post to XRechnung Endpoint**
    - *Type and Technical Role:* HTTP Request node targeting the XRechnung XML generation service.
    - *Configuration:* POST request to `https://us1.pdfgeneratorapi.com/api/v4/einvoice/xrechnung`. JSON body provides data payload, type (`UBL`), and output format (`base64`). Uses predefined credential type `pdfGeneratorApi`.
    - *Input/Output:* Connected from branch 1 of `Route by Invoice Format`; outputs to `Parse API Response`.
    - *Credentials:* `pdfGeneratorApi`
  - **Post to EN 16931 Endpoint**
    - *Type and Technical Role:* HTTP Request node targeting the general EN 16931 XML generation service.
    - *Configuration:* POST request to `https://us1.pdfgeneratorapi.com/api/v4/einvoice`. JSON body provides data payload, type (`UBL`), and output format (`base64`). Uses predefined credential type `pdfGeneratorApi`.
    - *Input/Output:* Connected from branch 2 of `Route by Invoice Format`; outputs to `Parse API Response`.
    - *Credentials:* `pdfGeneratorApi`
  - **Parse API Response**
    - *Type and Technical Role:* Code node processing items individually (`runOnceForEachItem`).
    - *Configuration:* Extracts the metadata object, determines MIME type from `content-type` headers (defaulting to `application/xml`), builds a file name based on the invoice number and extension, and isolates the base64 string.
    - *Input/Output:* Connected from all three HTTP Request generation nodes; outputs to `Convert Response to File`.

---

#### 2.6 File Conversion & Handoff
- **Overview:** Converts the decoded base64 document into a structured binary file format and passes it downstream to an integration placeholder.
- **Nodes Involved:**
  - `Convert Response to File` (`n8n-nodes-base.convertToFile`)
  - `Store or Send Generated File` (`n8n-nodes-base.noOp`)

- **Node Details:**
  - **Convert Response to File**
    - *Type and Technical Role:* Convert to File utility node.
    - *Configuration:* Operation set to `toBinary`, taking data from source property `data`. Sets dynamic expressions for `fileName` (`={{ $json.file_name }}`) and `mimeType` (`={{ $json.mime_type }}`).
    - *Input/Output:* Connected from `Parse API Response`; outputs to `Store or Send Generated File`.
  - **Store or Send Generated File**
    - *Type and Technical Role:* No-Operation placeholder node indicating where external delivery systems (e.g., Gmail, S3, Google Drive, or ERP integrations) should be connected.
    - *Configuration:* Default No-Op configuration.
    - *Input/Output:* Connected from `Convert Response to File`; has no outgoing connections.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Manual Trigger Workflow | `n8n-nodes-base.manualTrigger` | Initiates manual workflow runs for testing. | None | Prepare Sample Invoice, Fetch E-Invoice Schema | Generate compliant e-invoices (EN 16931, XRechnung, Factur-X) with PDF Generator API |
| Webhook Trigger for Invoice | `n8n-nodes-base.webhook` | Receives incoming external invoice webhook requests. | None | Prepare Webhook Invoice | Generate compliant e-invoices (EN 16931, XRechnung, Factur-X) with PDF Generator API |
| Prepare Sample Invoice | `n8n-nodes-base.set` | Provides static sample invoice data for manual runs. | Manual Trigger Workflow | Set Invoice Parameters | Generate compliant e-invoices (EN 16931, XRechnung, Factur-X) with PDF Generator API |
| Prepare Webhook Invoice | `n8n-nodes-base.set` | Extracts raw JSON body from incoming webhooks. | Webhook Trigger for Invoice | Set Invoice Parameters | Generate compliant e-invoices (EN 16931, XRechnung, Factur-X) with PDF Generator API |
| Fetch E-Invoice Schema | `n8n-nodes-base.httpRequest` | Retrieves the API schema definition reference. | Manual Trigger Workflow | None | Schema lookup |
| Set Invoice Parameters | `n8n-nodes-base.set` | Assigns global template, format, and profile settings. | Prepare Sample Invoice, Prepare Webhook Invoice | Map to UBL Invoice Format | Invoice intake and settings |
| Map to UBL Invoice Format | `n8n-nodes-base.code` | Transforms invoice data into Peppol BIS Billing 3.0 UBL. | Set Invoice Parameters | Route by Invoice Format | Build invoice payload |
| Route by Invoice Format | `n8n-nodes-base.switch` | Branches execution depending on target e-invoice format. | Map to UBL Invoice Format | Post to Factur-X Endpoint, Post to XRechnung Endpoint, Post to EN 16931 Endpoint | Choose output format |
| Post to Factur-X Endpoint | `n8n-nodes-base.httpRequest` | Generates a Factur-X PDF with embedded XML via API. | Route by Invoice Format | Parse API Response | Generate e-invoice document |
| Post to XRechnung Endpoint | `n8n-nodes-base.httpRequest` | Generates an XRechnung XML document via API. | Route by Invoice Format | Parse API Response | Generate e-invoice document |
| Post to EN 16931 Endpoint | `n8n-nodes-base.httpRequest` | Generates an EN 16931 XML document via API. | Route by Invoice Format | Parse API Response | Generate e-invoice document |
| Parse API Response | `n8n-nodes-base.code` | Extracts base64 document content and metadata. | Post to Factur-X Endpoint, Post to XRechnung Endpoint, Post to EN 16931 Endpoint | Convert Response to File | Parse API response |
| Convert Response to File | `n8n-nodes-base.convertToFile` | Transforms base64 string into a binary file. | Parse API Response | Store or Send Generated File | Create and hand off file |
| Store or Send Generated File | `n8n-nodes-base.noOp` | Placeholder for delivery, storage, or ERP handoff. | Convert Response to File | None | Create and hand off file |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow manually in n8n:

1. **Create Entry Triggers and Setup:**
   - Add a **Manual Trigger** node (`n8n-nodes-base.manualTrigger`) named `Manual Trigger Workflow`.
   - Add a **Webhook** node (`n8n-nodes-base.webhook`) named `Webhook Trigger for Invoice`. Set the HTTP method to `POST`, path to `e-invoice`, and response mode to `Last Node`.

2. **Add Data Preparation Nodes:**
   - Add a **Set** node (`n8n-nodes-base.set`) named `Prepare Sample Invoice`. Set mode to `raw` and populate the `jsonOutput` parameter with a sample invoice JSON object containing `format`, `invoice_number`, `issue_date`, `due_date`, `currency`, `seller`, `buyer`, and `lines`. Connect `Manual Trigger Workflow` to this node.
   - Add a **Set** node (`n8n-nodes-base.set`) named `Prepare Webhook Invoice`. Set mode to `raw` and configure the assignment expression as `={{ JSON.stringify($json.body) }}`. Connect `Webhook Trigger for Invoice` to this node.

3. **Add Global Configuration Node:**
   - Add a **Set** node (`n8n-nodes-base.set`) named `Set Invoice Parameters`.
   - Configure assignments to include:
     - `template_id` (number: `1656704`)
     - `default_format` (string: `"facturx"`)
     - `facturx_profile` (string: `"en16931"`)
   - Enable `Include Other Fields`. Connect both `Prepare Sample Invoice` and `Prepare Webhook Invoice` outputs to this node.

4. **Add UBL Mapping Logic (Code Node):**
   - Add a **Code** node (`n8n-nodes-base.code`) named `Map to UBL Invoice Format`.
   - Paste the JavaScript snippet that calculates item extensions, rounds currency values, generates VAT subtotals, maps supplier and customer party details, and outputs the structured UBL envelope under the `ubl` property.
   - Connect `Set Invoice Parameters` to this node.

5. **Add Format Routing Node:**
   - Add a **Switch** node (`n8n-nodes-base.switch`) named `Route by Invoice Format`.
   - Configure three output rules evaluating `={{ $json.format }}`:
     - Rule 1 (Factur-X): Equals `"facturx"`
     - Rule 2 (XRechnung): Equals `"xrechnung"`
     - Rule 3 (EN 16931): Equals `"en16931"`
   - Connect `Map to UBL Invoice Format` to this switch node.

6. **Add API Generation Nodes:**
   - Create a **PDF Generator API** credential with your API key, secret, and account email.
   - Add an **HTTP Request** node named `Post to Factur-X Endpoint`:
     - Method: `POST`, URL: `https://us1.pdfgeneratorapi.com/api/v4/einvoice/facturx`
     - Body content type: JSON. Specify body using expression: `={{ JSON.stringify({ template: { id: $json.template_id, data: $json.ubl }, profile: $json.profile, output: 'base64', name: $json.document_name }) }}`
     - Set authentication to predefined credential type `pdfGeneratorApi`.
     - Connect output 0 of `Route by Invoice Format` here.
   - Add an **HTTP Request** node named `Post to XRechnung Endpoint`:
     - Method: `POST`, URL: `https://us1.pdfgeneratorapi.com/api/v4/einvoice/xrechnung`
     - Body content type: JSON. Specify body using expression: `={{ JSON.stringify({ data: $json.ubl, type: 'UBL', output: 'base64' }) }}`
     - Set authentication to predefined credential type `pdfGeneratorApi`.
     - Connect output 1 of `Route by Invoice Format` here.
   - Add an **HTTP Request** node named `Post to EN 16931 Endpoint`:
     - Method: `POST`, URL: `https://us1.pdfgeneratorapi.com/api/v4/einvoice`
     - Body content type: JSON. Specify body using expression: `={{ JSON.stringify({ data: $json.ubl, type: 'UBL', output: 'base64' }) }}`
     - Set authentication to predefined credential type `pdfGeneratorApi`.
     - Connect output 2 of `Route by Invoice Format` here.

7. **Add Response Parsing and Conversion Nodes:**
   - Add a **Code** node (`n8n-nodes-base.code`) named `Parse API Response` configured for execution per item (`runOnceForEachItem`). Paste the parser code to extract MIME types, filenames, and base64 strings. Connect all three HTTP request nodes to this node.
   - Add a **Convert to File** node (`n8n-nodes-base.convertToFile`) named `Convert Response to File`:
     - Operation: `To Binary File`
     - Source Property: `data`
     - File Name: `={{ $json.file_name }}`
     - MIME Type: `={{ $json.mime_type }}`
     - Connect `Parse API Response` to this node.
   - Add a **No-Op** node (`n8n-nodes-base.noOp`) named `Store or Send Generated File` as the terminal step, connected from `Convert Response to File`.

8. **Optional Schema Node (Reference):**
   - Add an **HTTP Request** node named `Fetch E-Invoice Schema` (`disabled: true`), configured as a GET request to `https://us1.pdfgeneratorapi.com/api/v4/einvoice/schema` using the `pdfGeneratorApi` credential, connected from `Manual Trigger Workflow`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| PDF Generator API E-Invoice Documentation | [PDF Generator API Documentation](https://us1.pdfgeneratorapi.com/) |
| Peppol BIS Billing 3.0 Standard Reference | Peppol eDelivery Interoperability Network Specifications |
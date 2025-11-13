Convert HTML & PDF Files to PNG Images with CustomJS PDF Toolkit

https://n8nworkflows.xyz/workflows/convert-html---pdf-files-to-png-images-with-customjs-pdf-toolkit-3870


# Convert HTML & PDF Files to PNG Images with CustomJS PDF Toolkit

### 1. Workflow Overview

This n8n workflow demonstrates how to convert HTML and PDF files into PNG images using the CustomJS PDF Toolkit. It is designed for self-hosted n8n instances and leverages the CustomJS API for PDF generation and conversion. The workflow covers two main use cases:

- Generating a PDF document from raw HTML content and then converting that PDF into PNG images.
- Accepting a URL pointing to an existing PDF file, downloading it, and converting that PDF into PNG images.

The workflow is divided into three logical blocks:

- **1.1 Input Reception and Trigger:** Handles starting the workflow manually.
- **1.2 PDF Generation from HTML:** Converts provided HTML content into a PDF file using CustomJS API.
- **1.3 PDF Processing and Conversion to PNG:** Converts either the generated PDF or a PDF fetched via URL into PNG images using CustomJS API.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Trigger

**Overview:**  
This block initiates the workflow execution manually by user interaction, triggering both PDF generation from HTML and PDF conversion from an external URL.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)

**Node Details:**

- **When clicking ‘Test workflow’**  
  - *Type & Role:* Manual Trigger node; starts the workflow manually when user clicks "Execute Workflow".  
  - *Configuration:* No parameters; default manual trigger.  
  - *Input/Output:* No input; outputs trigger data to two parallel paths — one to HTML to PDF, one to Set PDF URL.  
  - *Version:* Standard, no special version requirements.  
  - *Edge Cases:* None significant; user must manually trigger workflow.  
  - *Sub-Workflow:* None.

---

#### 2.2 PDF Generation from HTML

**Overview:**  
This block converts raw HTML content into a PDF document using the CustomJS HTML to PDF node, then sends the PDF output to conversion into PNG images.

**Nodes Involved:**  
- HTML to PDF  
- Convert PDF into PNG

**Node Details:**

- **HTML to PDF**  
  - *Type & Role:* `@custom-js/n8n-nodes-pdf-toolkit.html2Pdf` node; converts HTML input string into a PDF file.  
  - *Configuration:*  
    - `htmlInput`: Static HTML string `<h1>Hello World</h1>`.  
    - Credentials: Connected to CustomJS API credentials named "CustomJS account" for authentication.  
  - *Input/Output:*  
    - Input: Triggered from manual trigger node.  
    - Output: PDF file data passed to "Convert PDF into PNG" node.  
  - *Version:* v1 of CustomJS node.  
  - *Edge Cases:*  
    - Invalid or empty HTML input may cause PDF generation failure.  
    - API key or network issues can cause auth or connectivity errors.  
  - *Sub-Workflow:* None.

- **Convert PDF into PNG**  
  - *Type & Role:* `@custom-js/n8n-nodes-pdf-toolkit.PdfToPng` node; converts PDF binary data from previous node to PNG images.  
  - *Configuration:* Default parameters; uses PDF binary data directly from previous node.  
  - *Credentials:* Uses same CustomJS API credentials.  
  - *Input/Output:*  
    - Input: PDF data from "HTML to PDF" node.  
    - Output: PNG image files generated.  
  - *Version:* v1.  
  - *Edge Cases:*  
    - Conversion errors if PDF data is corrupted or empty.  
    - API or network failures.  
  - *Sub-Workflow:* None.

---

#### 2.3 PDF Processing and Conversion to PNG from URL

**Overview:**  
This block handles a PDF file located at a specific URL by preparing the URL in a Code node, then converting the PDF from that URL into PNG images.

**Nodes Involved:**  
- Set PDF URL (Code)  
- Convert PDF into PNG1

**Node Details:**

- **Set PDF URL**  
  - *Type & Role:* n8n Code node; outputs JSON object with a `path` property containing the PDF URL string.  
  - *Configuration:*  
    - Inline JavaScript returns JSON: `{ "path": "https://www.nlbk.niedersachsen.de/download/164891/Test-pdf_3.pdf.pdf" }`.  
  - *Input/Output:*  
    - Input: Triggered from manual trigger node.  
    - Output: JSON data with PDF URL passed to "Convert PDF into PNG1".  
  - *Version:* Version 2 of Code node, supports modern syntax.  
  - *Edge Cases:*  
    - URL must be accessible and valid; inaccessible URLs cause failures downstream.  
  - *Sub-Workflow:* None.

- **Convert PDF into PNG1**  
  - *Type & Role:* `@custom-js/n8n-nodes-pdf-toolkit.PdfToPng` node; converts PDF from URL to PNG images.  
  - *Configuration:*  
    - `resource` set to `"url"` indicating input is a URL.  
    - `field_name` expression set to `{{$json.path}}` to dynamically use URL from previous node.  
  - *Credentials:* Uses CustomJS API key for conversion calls.  
  - *Input/Output:*  
    - Input: URL from "Set PDF URL" node.  
    - Output: PNG images generated from the PDF at the URL.  
  - *Version:* v1.  
  - *Edge Cases:*  
    - Invalid URL or inaccessible PDF causes errors.  
    - Network or API failures possible.  
  - *Sub-Workflow:* None.

---

### 3. Summary Table

| Node Name                | Node Type                                  | Functional Role                         | Input Node(s)                | Output Node(s)            | Sticky Note                                                                                               |
|--------------------------|--------------------------------------------|---------------------------------------|-----------------------------|---------------------------|----------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                            | Starts workflow manually               |                             | HTML to PDF, Set PDF URL  |                                                                                                          |
| HTML to PDF              | @custom-js/n8n-nodes-pdf-toolkit.html2Pdf | Converts HTML string to PDF            | When clicking ‘Test workflow’ | Convert PDF into PNG       | ### HTML to PDF<br>- Request HTML Data.<br>- Convert HTML to PDF.                                        |
| Convert PDF into PNG     | @custom-js/n8n-nodes-pdf-toolkit.PdfToPng | Converts PDF binary to PNG images      | HTML to PDF                 |                           | ### Convert PDF into PNG <br>- Convert the generated PNG from PDF                                        |
| Set PDF URL              | Code                                        | Sets PDF URL as JSON property `path`  | When clicking ‘Test workflow’ | Convert PDF into PNG1      | ### Set PDF URL<br>- Request PDF from URL                                                               |
| Convert PDF into PNG1    | @custom-js/n8n-nodes-pdf-toolkit.PdfToPng | Converts PDF from URL to PNG images    | Set PDF URL                 |                           | ### Convert PDF into PNG<br>- Convert the generated PNG from PDF                                        |
| Sticky Note              | Sticky Note                                 | Documentation - HTML to PDF block      |                             |                           | ### HTML to PDF<br>- Request HTML Data.<br>- Convert HTML to PDF.                                        |
| Sticky Note1             | Sticky Note                                 | Documentation - Convert PDF to PNG     |                             |                           | ### Convert PDF into PNG <br>- Convert the generated PNG from PDF                                        |
| Sticky Note2             | Sticky Note                                 | Documentation - Set PDF URL             |                             |                           | ### Set PDF URL<br>- Request PDF from URL                                                               |
| Sticky Note3             | Sticky Note                                 | Documentation - Convert PDF to PNG     |                             |                           | ### Convert PDF into PNG<br>- Convert the generated PNG from PDF                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a Manual Trigger node named `When clicking ‘Test workflow’`.  
   - Leave default parameters; no inputs or credentials required.

2. **Create HTML to PDF Node**  
   - Add a node of type `@custom-js/n8n-nodes-pdf-toolkit.html2Pdf` named `HTML to PDF`.  
   - Set parameter `htmlInput` to `<h1>Hello World</h1>`.  
   - Assign credentials: Select or create CustomJS API credential with your API key.  
   - Connect `When clicking ‘Test workflow’` → `HTML to PDF`.

3. **Create Convert PDF into PNG Node**  
   - Add a node of type `@custom-js/n8n-nodes-pdf-toolkit.PdfToPng` named `Convert PDF into PNG`.  
   - Use default parameters (it will use incoming PDF binary data).  
   - Assign the same CustomJS API credentials as above.  
   - Connect `HTML to PDF` → `Convert PDF into PNG`.

4. **Create Code Node to Set PDF URL**  
   - Add a Code node named `Set PDF URL`.  
   - Use JavaScript code to output JSON with the PDF path URL:  
     ```javascript
     return {
       json: {
         path: "https://www.nlbk.niedersachsen.de/download/164891/Test-pdf_3.pdf.pdf"
       }
     };
     ```  
   - Connect `When clicking ‘Test workflow’` → `Set PDF URL`.

5. **Create Convert PDF into PNG1 Node**  
   - Add a node of type `@custom-js/n8n-nodes-pdf-toolkit.PdfToPng` named `Convert PDF into PNG1`.  
   - Set parameter `resource` to `"url"`.  
   - Set `field_name` to the expression `{{$json.path}}` to dynamically use the URL from the Code node.  
   - Assign the same CustomJS API credentials.  
   - Connect `Set PDF URL` → `Convert PDF into PNG1`.

6. **Add Sticky Notes (Optional)**  
   - Add sticky notes near relevant nodes to document the workflow logic as per the original layout, e.g., describing each block's purpose.

7. **Credential Setup**  
   - Ensure you have CustomJS API credentials created under n8n credentials with your valid API key from https://www.customjs.space.  
   - Use these credentials on all CustomJS PDF Toolkit nodes.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                          | Context or Link                                                                      |
|-------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------|
| Community nodes like `@custom-js/n8n-nodes-pdf-toolkit` require self-hosted n8n instances to install and run.                                         | https://www.npmjs.com/package/@custom-js/n8n-nodes-pdf-toolkit                       |
| Obtain API key by signing up at CustomJS platform, then navigate to profile to reveal API key.                                                        | https://www.customjs.space                                                           |
| You can replace the Manual Trigger with a Webhook node for automated triggering and output results accordingly.                                      | Workflow design suggestion                                                          |
| Sticky notes provide helpful high-level descriptions for each block to aid comprehension and future maintenance.                                     | See Sticky Note nodes in workflow                                                   |

---

This documentation fully describes the workflow’s structure, node details, dependencies, and instructions to reproduce it in n8n without any external references.
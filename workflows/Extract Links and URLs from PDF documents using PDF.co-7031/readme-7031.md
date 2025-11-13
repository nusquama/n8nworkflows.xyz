Extract Links and URLs from PDF documents using PDF.co

https://n8nworkflows.xyz/workflows/extract-links-and-urls-from-pdf-documents-using-pdf-co-7031


# Extract Links and URLs from PDF documents using PDF.co

### 1. Workflow Overview

This n8n workflow is designed to extract all links and URLs from PDF documents using the PDF.co API. It targets use cases where users upload PDF files and require automated extraction of embedded hyperlinks as plain URLs for further processing or analysis.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Handles receiving the PDF file from the user via a web form.
- **1.2 Upload to PDF.co:** Uploads the received PDF file to PDF.co to obtain a usable URL.
- **1.3 PDF to HTML Conversion:** Converts the uploaded PDF from its URL to an HTML format using PDF.co.
- **1.4 Retrieve HTML Content:** Fetches the HTML content from the conversion result URL.
- **1.5 Extract URLs:** Processes the HTML content with JavaScript code to extract all URLs using regex.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block starts the workflow by providing a web form for users to upload a single PDF file.

- **Nodes Involved:**  
  - Load PDF

- **Node Details:**

  - **Load PDF**  
    - Type: `Form Trigger`  
    - Role: Receives user input in the form of a PDF file upload.  
    - Configuration:  
      - Form Title: "pdf"  
      - Form Field: Single file upload accepting `.pdf` files only, named "data"  
    - Input: N/A (trigger node)  
    - Output: Binary data of uploaded PDF file  
    - Edge Cases:  
      - Uploading non-PDF files is prevented by accept filter but can fail if bypassed.  
      - Large files may cause timeout or memory issues depending on environment.  
      - Form trigger requires the webhook to be publicly accessible.  
    - Version: 2.2

---

#### 2.2 Upload to PDF.co

- **Overview:**  
  Uploads the PDF binary data to PDF.co to get a hosted URL for the file, enabling API-based processing.

- **Nodes Involved:**  
  - Upload

- **Node Details:**

  - **Upload**  
    - Type: `PDFco Api`  
    - Role: Upload PDF binary data received from the form to PDF.co file storage.  
    - Configuration:  
      - Operation: "Upload File to PDF.co"  
      - Uses binary data input (the uploaded PDF)  
    - Credentials: PDF.co API credentials required and configured.  
    - Input: Binary PDF data from "Load PDF" node  
    - Output: JSON containing URL to the uploaded PDF file (property: `url`)  
    - Edge Cases:  
      - Authentication failure due to invalid API key.  
      - Network timeouts or API rate limits.  
      - Large file size limits imposed by PDF.co.  
    - Version: 1

---

#### 2.3 PDF to HTML Conversion

- **Overview:**  
  Converts the uploaded PDF file (accessed by URL) into HTML format to facilitate link extraction.

- **Nodes Involved:**  
  - PDF to HTML

- **Node Details:**

  - **PDF to HTML**  
    - Type: `PDFco Api`  
    - Role: Converts PDF file from URL to HTML format via PDF.co service.  
    - Configuration:  
      - Operation: "Convert from PDF"  
      - URL: Uses the `url` property from the previous node output (`{{$json.url}}`)  
      - No advanced options configured  
    - Credentials: PDF.co API credentials required and configured.  
    - Input: JSON with `url` from "Upload" node  
    - Output: JSON containing URL to the generated HTML file (`url`)  
    - Edge Cases:  
      - Invalid or expired URL from upload step.  
      - Conversion errors due to corrupted PDF.  
      - API rate limits or failures.  
    - Version: 1

---

#### 2.4 Retrieve HTML Content

- **Overview:**  
  Downloads the HTML content from the PDF to HTML conversion URL to prepare for link extraction.

- **Nodes Involved:**  
  - Get HTML

- **Node Details:**

  - **Get HTML**  
    - Type: `HTTP Request`  
    - Role: Fetches the raw HTML content from the URL provided by the "PDF to HTML" node.  
    - Configuration:  
      - URL: Dynamically set from the `url` field of previous node output (`{{$json.url}}`)  
      - Method: Default GET  
      - No additional headers or authentication  
    - Input: JSON with `url` from "PDF to HTML" node  
    - Output: JSON containing HTML content under the `data` property  
    - Edge Cases:  
      - URL might be invalid or expired.  
      - Network issues or timeouts.  
      - Large HTML payloads may require memory considerations.  
    - Version: 4.2

---

#### 2.5 Extract URLs

- **Overview:**  
  Uses JavaScript code to parse the HTML content and extract all embedded URLs using a regular expression.

- **Nodes Involved:**  
  - Code1

- **Node Details:**

  - **Code1**  
    - Type: `Code` (JavaScript)  
    - Role: Processes input data (HTML content) to find and output all URLs found.  
    - Configuration:  
      - Iterates over all incoming items.  
      - Reads the `data` property containing HTML text.  
      - Uses regex to match HTTP/HTTPS and www-prefixed URLs.  
      - For each URL found, creates a new output item with `{ url: <matched_url> }`.  
    - Key Expression:  
      - Regex: `/(https?:\/\/[^\s]+)|(www\.[^\s]+)/gi`  
    - Input: JSON with `data` from "Get HTML" node  
    - Output: Multiple items each containing one extracted URL  
    - Edge Cases:  
      - No URLs found results in empty output array.  
      - HTML content might contain malformed or partial URLs.  
      - Regex might catch false positives if text contains URL-like strings.  
    - Version: 2

---

### 3. Summary Table

| Node Name    | Node Type          | Functional Role                 | Input Node(s)   | Output Node(s) | Sticky Note Content          |
|--------------|--------------------|--------------------------------|-----------------|----------------|-----------------------------|
| Load PDF     | Form Trigger       | Receive PDF file upload         | -               | Upload         | ## Load PDF                 |
| Upload       | PDFco Api          | Upload PDF to PDF.co            | Load PDF        | PDF to HTML    | ## Upload to PDF.CO         |
| PDF to HTML  | PDFco Api          | Convert PDF to HTML             | Upload          | Get HTML       | ## PDF to HTML              |
| Get HTML     | HTTP Request       | Fetch HTML content from URL    | PDF to HTML     | Code1          | ## Get HTML                 |
| Code1        | Code (JavaScript)  | Extract URLs from HTML content  | Get HTML        | -              | ## Get URL's                |
| Sticky Note  | Sticky Note        | Annotation                    | -               | -              | ## Load PDF                 |
| Sticky Note1 | Sticky Note        | Annotation                    | -               | -              | ## Upload to PDF.CO         |
| Sticky Note2 | Sticky Note        | Annotation                    | -               | -              | ## PDF to HTML              |
| Sticky Note3 | Sticky Note        | Annotation                    | -               | -              | ## Get HTML                 |
| Sticky Note4 | Sticky Note        | Annotation                    | -               | -              | ## Get URL's                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Form Trigger node**  
   - Name: `Load PDF`  
   - Set form title to `"pdf"`  
   - Add a form field: type `file`, label `"data"`, accept only `.pdf` files, allow single file upload.  
   - This node acts as the webhook entry point.

3. **Add a PDF.co API node**  
   - Name: `Upload`  
   - Set operation to `"Upload File to PDF.co"`  
   - Enable binary data input to upload the PDF from the form trigger.  
   - Connect `Load PDF` → `Upload`  
   - Set up PDF.co credentials (API key) in n8n credentials manager.  
   - Use the binary file from `Load PDF` as input.

4. **Add another PDF.co API node**  
   - Name: `PDF to HTML`  
   - Set operation to `"Convert from PDF"`  
   - Set the URL parameter to `{{$json.url}}` referencing the URL output from the `Upload` node.  
   - Connect `Upload` → `PDF to HTML`  
   - Use the same PDF.co credentials.

5. **Add an HTTP Request node**  
   - Name: `Get HTML`  
   - Set method to `GET`  
   - Set URL to `{{$json.url}}` from the `PDF to HTML` node output.  
   - Connect `PDF to HTML` → `Get HTML`

6. **Add a Code node (JavaScript)**  
   - Name: `Code1`  
   - Paste the following JavaScript code to extract URLs:

   ```javascript
   const resultados = [];

   for (const item of $input.all()) {
     const texto = item.json.data || '';
     const regexUrl = /(https?:\/\/[^\s]+)|(www\.[^\s]+)/gi;
     const urls = texto.match(regexUrl) || [];

     for (const url of urls) {
       resultados.push({ json: { url } });
     }
   }

   return resultados;
   ```

   - Connect `Get HTML` → `Code1`

7. **Add Sticky Notes** for documentation purposes (optional but recommended):  
   - Attach each sticky note near the corresponding node with titles:  
     - Load PDF  
     - Upload to PDF.CO  
     - PDF to HTML  
     - Get HTML  
     - Get URL's

8. **Activate and test the workflow** by sending a PDF file to the webhook URL generated by the `Load PDF` form trigger.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                           |
|-----------------------------------------------------------------------------------------------------|-----------------------------------------------------------|
| This workflow requires a valid PDF.co API key; sign up at https://pdf.co to obtain one.             | PDF.co official website                                    |
| The workflow relies on publicly accessible webhook URL for the form trigger node.                    | n8n webhook documentation                                 |
| Regex used extracts URLs starting with http, https, or www. It may include some false positives.    | Regex details can be customized based on specific needs.  |
| Large PDF files or complex PDFs may affect performance or cause API limits to be hit.                | Consult PDF.co API limits documentation                    |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.
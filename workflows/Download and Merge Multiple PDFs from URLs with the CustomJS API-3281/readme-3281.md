Download and Merge Multiple PDFs from URLs with the CustomJS API

https://n8nworkflows.xyz/workflows/download-and-merge-multiple-pdfs-from-urls-with-the-customjs-api-3281


# Download and Merge Multiple PDFs from URLs with the CustomJS API

### 1. Workflow Overview

This workflow automates the process of downloading multiple PDF files from specified public URLs, merging them into a single PDF document, and saving the result locally. It is designed for use cases such as bundling reports, invoices, or any document sets sourced externally without requiring custom code development.

The workflow is logically divided into the following blocks:

- **1.1 Input Initialization:** Defines the array of PDF URLs to process.
- **1.2 Splitting and Downloading PDFs:** Splits the array to handle each URL individually and downloads each PDF.
- **1.3 Merging PDFs:** Combines all downloaded PDFs into one file using the CustomJS PDF Toolkit.
- **1.4 Writing Output:** Saves the merged PDF file to disk.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Initialization

- **Overview:**  
  This block initializes the workflow by defining the list of PDF URLs to be processed. It starts the workflow manually and outputs the array of URLs.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - PDF Array (Code node)

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Starts the workflow manually for testing or execution.  
    - Configuration: No parameters; triggers downstream nodes on manual activation.  
    - Inputs: None  
    - Outputs: Connected to PDF Array node  
    - Edge Cases: None specific; user must manually trigger.

  - **PDF Array**  
    - Type: Code (JavaScript)  
    - Role: Returns an array of PDF URLs as workflow data.  
    - Configuration: Returns a JSON object with a key `data` containing an array of two PDF URLs:
      ```js
      return { data: [
        "https://www.intewa.com/fileadmin/documents/pdf-file.pdf",
        "https://www.w3.org/WAI/ER/tests/xhtml/testfiles/resources/pdf/dummy.pdf"
      ]}
      ```
    - Inputs: From Manual Trigger  
    - Outputs: Passes array to Split Out node  
    - Edge Cases: If URLs are invalid or empty, downstream nodes may fail or produce empty output.

#### 2.2 Splitting and Downloading PDFs

- **Overview:**  
  This block splits the array of URLs so each URL is processed individually, then downloads each PDF file via HTTP requests.

- **Nodes Involved:**  
  - Split Out  
  - HTTP Request1

- **Node Details:**

  - **Split Out**  
    - Type: Split Out  
    - Role: Splits the array of URLs into individual items for parallel processing.  
    - Configuration: Splits on the field `data` which contains the array of URLs.  
    - Inputs: From PDF Array node  
    - Outputs: Each output item contains one URL in `data` field, connected to HTTP Request1  
    - Edge Cases: If the array is empty, no outputs are generated.

  - **HTTP Request1**  
    - Type: HTTP Request  
    - Role: Downloads the PDF file from the URL provided in each item.  
    - Configuration:  
      - URL: Expression `={{ $json.data }}` dynamically uses the URL from the current item.  
      - Method: GET (default)  
      - Response Format: Binary (default for file download)  
    - Inputs: From Split Out node (one URL per item)  
    - Outputs: Binary data of the downloaded PDF, passed to Merge PDF node  
    - Edge Cases:  
      - HTTP errors (404, 403, timeout) may cause failures.  
      - Invalid URLs or network issues may interrupt the download.  
      - Large files may cause memory or timeout issues.

#### 2.3 Merging PDFs

- **Overview:**  
  This block merges all the downloaded PDF files into a single PDF document using the CustomJS PDF Toolkit node.

- **Nodes Involved:**  
  - Merge PDF  
  - Read/Write Files from Disk2  
  - Read/Write Files from Disk3

- **Node Details:**

  - **Merge PDF**  
    - Type: `@custom-js/n8n-nodes-pdf-toolkit.mergePdfs` (Community node)  
    - Role: Merges multiple PDF binary inputs into one PDF file.  
    - Configuration:  
      - No additional parameters required; it merges all incoming PDFs.  
      - Credentials: Uses CustomJS API credentials (API key) configured in n8n.  
    - Inputs: Receives binary PDF data from HTTP Request1 node (multiple inputs)  
    - Outputs: Merged PDF binary data forwarded to Read/Write Files from Disk2  
    - Version-specific: Requires installation of the CustomJS PDF Toolkit community node and valid API credentials.  
    - Edge Cases:  
      - API key invalid or missing causes authentication errors.  
      - Corrupted or invalid PDF inputs may cause merge failures.  
      - Rate limits or API downtime on CustomJS platform.

  - **Read/Write Files from Disk2**  
    - Type: Read/Write File  
    - Role: Writes the merged PDF binary data to a file named `test.pdf` on disk.  
    - Configuration:  
      - Operation: Write  
      - File Name: `test.pdf` (fixed)  
    - Inputs: From Merge PDF node  
    - Outputs: Passes file reference to Read/Write Files from Disk3  
    - Edge Cases:  
      - File system permissions or path issues may cause write failures.  
      - Disk space limitations.

  - **Read/Write Files from Disk3**  
    - Type: Read/Write File  
    - Role: Reads the saved `test.pdf` file from disk (could be used for further processing or verification).  
    - Configuration:  
      - Operation: Read  
      - File Selector: `test.pdf` (fixed)  
    - Inputs: From Read/Write Files from Disk2  
    - Outputs: None connected downstream (end of workflow)  
    - Edge Cases:  
      - File not found if write failed.  
      - File access permissions.

---

### 3. Summary Table

| Node Name                  | Node Type                          | Functional Role                         | Input Node(s)                  | Output Node(s)               | Sticky Note                                                                                         |
|----------------------------|----------------------------------|---------------------------------------|-------------------------------|-----------------------------|---------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                   | Starts the workflow manually           | None                          | PDF Array                   |                                                                                                   |
| PDF Array                  | Code                             | Defines array of PDF URLs               | When clicking ‘Test workflow’ | Split Out                   |                                                                                                   |
| Split Out                  | Split Out                        | Splits array into individual URLs      | PDF Array                     | HTTP Request1               |                                                                                                   |
| HTTP Request1              | HTTP Request                    | Downloads each PDF from URL             | Split Out                     | Merge PDF                   |                                                                                                   |
| Merge PDF                  | @custom-js/n8n-nodes-pdf-toolkit.mergePdfs | Merges multiple PDFs into one          | HTTP Request1                 | Read/Write Files from Disk2 | Requires CustomJS API credentials; community node only installable on self-hosted n8n instances.  |
| Read/Write Files from Disk2 | Read/Write File                  | Writes merged PDF to disk               | Merge PDF                     | Read/Write Files from Disk3 |                                                                                                   |
| Read/Write Files from Disk3 | Read/Write File                  | Reads saved merged PDF from disk        | Read/Write Files from Disk2   | None                        |                                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node**  
   - Type: Manual Trigger  
   - Purpose: To start the workflow manually.

2. **Create Code node named "PDF Array"**  
   - Type: Code (JavaScript)  
   - Configuration:  
     ```js
     return { data: [
       "https://www.intewa.com/fileadmin/documents/pdf-file.pdf",
       "https://www.w3.org/WAI/ER/tests/xhtml/testfiles/resources/pdf/dummy.pdf"
     ]}
     ```  
   - Connect Manual Trigger output to this node.

3. **Create Split Out node**  
   - Type: Split Out  
   - Parameters:  
     - Field to split out: `data`  
   - Connect PDF Array output to this node.

4. **Create HTTP Request node named "HTTP Request1"**  
   - Type: HTTP Request  
   - Parameters:  
     - URL: Use expression `={{ $json.data }}` to dynamically get the URL from each split item.  
     - Method: GET (default)  
     - Response Format: Binary (default)  
   - Connect Split Out output to this node.

5. **Install and configure CustomJS PDF Toolkit community node**  
   - Install `@custom-js/n8n-nodes-pdf-toolkit` if not already installed.  
   - Create credentials for CustomJS API:  
     - Obtain API key from https://www.customjs.space (sign up, profile page, "Show" API key).  
     - Add credentials in n8n under CustomJS account type with the API key.

6. **Create Merge PDF node**  
   - Type: `@custom-js/n8n-nodes-pdf-toolkit.mergePdfs`  
   - Credentials: Select the CustomJS API credentials created above.  
   - Connect HTTP Request1 output to this node.

7. **Create Read/Write File node named "Read/Write Files from Disk2"**  
   - Type: Read/Write File  
   - Parameters:  
     - Operation: Write  
     - File Name: `test.pdf`  
   - Connect Merge PDF output to this node.

8. **Create Read/Write File node named "Read/Write Files from Disk3"**  
   - Type: Read/Write File  
   - Parameters:  
     - Operation: Read  
     - File Selector: `test.pdf`  
   - Connect Read/Write Files from Disk2 output to this node.

9. **Save and activate the workflow**  
   - Test by clicking the manual trigger button.  
   - The workflow downloads the PDFs, merges them, writes the merged file to disk, and reads it back.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                                                                     |
|-------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This workflow requires a free CustomJS account and API key.                                     | https://www.customjs.space                                                                         |
| CustomJS API key can be found on the user profile page after signing up on CustomJS platform.  | https://www.customjs.space/profile                                                                |
| Community nodes like CustomJS PDF Toolkit require self-hosted n8n instances to install.         | https://docs.n8n.io/integrations/community-nodes/                                                 |
| This workflow can be adapted to trigger via webhook and respond with the merged PDF instead of manual trigger and file write. |                                                                                                   |
| PDF Toolkit node documentation and npm package: [@custom-js/n8n-nodes-pdf-toolkit](https://www.npmjs.com/package/@custom-js/n8n-nodes-pdf-toolkit) |                                                                                                   |

---

This document fully describes the workflow "Download and Merge Multiple PDFs from URLs with the CustomJS API" for understanding, reproduction, and modification.
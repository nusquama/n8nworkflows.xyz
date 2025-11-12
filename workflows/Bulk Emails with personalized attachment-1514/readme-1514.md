Bulk Emails with personalized attachment

https://n8nworkflows.xyz/workflows/bulk-emails-with-personalized-attachment-1514


# Bulk Emails with personalized attachment

### 1. Workflow Overview

This workflow automates the process of sending personalized bulk emails with respective attachments. It is designed for use cases where a user needs to distribute customized documents (e.g., certificates) to multiple recipients, with each email containing a unique attachment matching the recipient.

The workflow logic is structured into the following blocks:

- **1.1 Input Reception**: Manual trigger to start the workflow.
- **1.2 Data Preparation**: Reading and parsing a CSV file containing recipient emails and attachment file names.
- **1.3 Batch Processing**: Splitting the parsed data into manageable batches for processing.
- **1.4 Attachment Retrieval**: Reading the file corresponding to each recipient's attachment.
- **1.5 Email Sending**: Sending an email to each recipient with the personalized attachment.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block initiates the workflow execution manually, allowing the user to trigger the bulk email sending process.

- **Nodes Involved:**  
  - On clicking 'execute'

- **Node Details:**  

  - **On clicking 'execute'**  
    - Type: **Manual Trigger**  
    - Role: Starts the workflow manually on user command.  
    - Configuration: Default manual trigger with no additional parameters.  
    - Inputs: None  
    - Outputs: Starts the workflow by passing an empty trigger event.  
    - Edge Cases: None typical, except the workflow will not run unless manually triggered.  
    - Sub-workflow: None

#### 2.2 Data Preparation

- **Overview:**  
  This block reads the CSV file containing the list of email addresses and corresponding file names. It parses the CSV to prepare structured data for batch processing.

- **Nodes Involved:**  
  - Read Binary File  
  - Spreadsheet File

- **Node Details:**  

  - **Read Binary File**  
    - Type: **Read Binary File**  
    - Role: Reads the CSV file from the local filesystem as binary data.  
    - Configuration:  
      - File path set to `/home/shashikanth/Documents/Cert-Gen-Test/data.csv`  
      - Data stored in binary property named `csv`.  
    - Inputs: Triggered by manual trigger node.  
    - Outputs: Binary data containing CSV content.  
    - Edge Cases:  
      - File not found or inaccessible path causes failure.  
      - Permission issues may occur on certain OS environments.  
    - Sub-workflow: None

  - **Spreadsheet File**  
    - Type: **Spreadsheet File**  
    - Role: Parses the binary CSV data into JSON objects.  
    - Configuration:  
      - Header row enabled to use CSV first row as keys.  
      - Reads from binary property `csv`.  
    - Inputs: Receives binary CSV data from previous node.  
    - Outputs: JSON array with each row as an object containing keys like `email` and `name`.  
    - Edge Cases:  
      - Malformed CSV can cause parsing errors.  
      - Missing headers or mismatched columns will impact downstream nodes.  
    - Sub-workflow: None

#### 2.3 Batch Processing

- **Overview:**  
  This block splits the parsed JSON array into batches of fixed size (5) to process them incrementally, preventing resource overload.

- **Nodes Involved:**  
  - SplitInBatches

- **Node Details:**  

  - **SplitInBatches**  
    - Type: **SplitInBatches**  
    - Role: Divides the list of recipients into batches of 5 for sequential processing.  
    - Configuration:  
      - Batch size set to 5.  
      - Reset option disabled to preserve batch state across executions.  
    - Inputs: JSON array of recipients from the Spreadsheet File node.  
    - Outputs: Emits batch of 5 recipient objects per execution cycle.  
    - Edge Cases:  
      - Batch size too large may cause memory issues.  
      - If input array empty, no batches emitted.  
    - Sub-workflow: None

#### 2.4 Attachment Retrieval

- **Overview:**  
  This block reads the personalized attachment file (PNG image) for each recipient based on the file name specified in the CSV.

- **Nodes Involved:**  
  - Read Binary File1

- **Node Details:**  

  - **Read Binary File1**  
    - Type: **Read Binary File**  
    - Role: Reads the attachment image file dynamically based on the current recipient's file name.  
    - Configuration:  
      - File path expression: `/home/shashikanth/Documents/Cert-Gen-Test/generator-output/{{$json["name"]}}.png`  
      - Reads the PNG file corresponding to the `name` field from the current batch item.  
    - Inputs: Receives a batch item (recipient JSON) from SplitInBatches.  
    - Outputs: Binary data of the attachment file.  
    - Edge Cases:  
      - File might not exist or path may be incorrect, causing errors.  
      - Filename must match exactly and be safe for file path usage.  
    - Sub-workflow: None

#### 2.5 Email Sending

- **Overview:**  
  This block sends an email with the personalized attachment to each recipient using SMTP credentials.

- **Nodes Involved:**  
  - Send Email

- **Node Details:**  

  - **Send Email**  
    - Type: **Email Send**  
    - Role: Sends an email with the predefined subject and attachment to the recipient email address.  
    - Configuration:  
      - Subject: "Certificate For Course"  
      - Recipient email: dynamically set from current batch item’s `email` field (`={{$node["SplitInBatches"].json["email"]}}`)  
      - Sender email: `bhavabhuthi@riseup.net`  
      - Attachments: uses binary property `data` (binary content from previous node)  
      - Allows unauthorized certificates (useful for testing or self-signed SMTP servers)  
    - Credentials: SMTP account configured with identifier "1"  
    - Inputs: Binary data of attachment from Read Binary File1 node.  
    - Outputs: Email sent status.  
    - Edge Cases:  
      - Authentication failure if SMTP credentials are invalid.  
      - Email sending failure due to server issues or invalid recipient addresses.  
      - Attachment binary must be correctly formatted, otherwise email may be rejected.  
    - Sub-workflow: None

---

### 3. Summary Table

| Node Name           | Node Type           | Functional Role                      | Input Node(s)             | Output Node(s)        | Sticky Note                                                                                 |
|---------------------|---------------------|------------------------------------|---------------------------|-----------------------|---------------------------------------------------------------------------------------------|
| On clicking 'execute'| Manual Trigger      | Initiates the workflow manually    | -                         | Read Binary File       |                                                                                             |
| Read Binary File     | Read Binary File    | Reads CSV file as binary data      | On clicking 'execute'      | Spreadsheet File       |                                                                                             |
| Spreadsheet File     | Spreadsheet File    | Parses CSV binary to JSON           | Read Binary File           | SplitInBatches         |                                                                                             |
| SplitInBatches       | Split In Batches    | Processes data in batches of 5     | Spreadsheet File           | Read Binary File1      |                                                                                             |
| Read Binary File1    | Read Binary File    | Reads personalized attachment file | SplitInBatches             | Send Email             |                                                                                             |
| Send Email           | Email Send          | Sends personalized email with attachment | Read Binary File1          | -                     |                                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the manual trigger node:**  
   - Add node: Manual Trigger  
   - Name: `On clicking 'execute'`  
   - Leave default parameters.

2. **Add Read Binary File node to read CSV:**  
   - Add node: Read Binary File  
   - Name: `Read Binary File`  
   - Set `File Path` to `/home/shashikanth/Documents/Cert-Gen-Test/data.csv`  
   - Set `Data Property Name` to `csv`.  
   - Connect output of Manual Trigger to this node.

3. **Add Spreadsheet File node to parse CSV:**  
   - Add node: Spreadsheet File  
   - Name: `Spreadsheet File`  
   - Enable `Header Row` option to true.  
   - Set `Binary Property Name` to `csv`.  
   - Connect output of `Read Binary File` to this node.

4. **Add SplitInBatches node:**  
   - Add node: SplitInBatches  
   - Name: `SplitInBatches`  
   - Set `Batch Size` to 5.  
   - Disable `Reset` option (leave unchecked).  
   - Connect output of `Spreadsheet File` to this node.

5. **Add Read Binary File node to read attachment:**  
   - Add node: Read Binary File  
   - Name: `Read Binary File1`  
   - Set `File Path` with expression:  
     `/home/shashikanth/Documents/Cert-Gen-Test/generator-output/{{$json["name"]}}.png`  
   - Connect output of `SplitInBatches` to this node.

6. **Add Send Email node:**  
   - Add node: Email Send  
   - Name: `Send Email`  
   - Configure SMTP credentials using your SMTP account (e.g., Gmail, Outlook) with OAuth2 or username/password.  
   - Set `From Email` to `bhavabhuthi@riseup.net`.  
   - Set `To Email` to expression: `={{$node["SplitInBatches"].json["email"]}}`  
   - Set `Subject` to `Certificate For Course`.  
   - Set `Attachments` to use the binary data property named `data` (default output from Read Binary File1).  
   - Enable option `Allow Unauthorized Certificates` if needed (typically for testing).  
   - Connect output of `Read Binary File1` to this node.

7. **Validate and test:**  
   - Ensure all nodes have correct connections:  
     Manual Trigger → Read Binary File → Spreadsheet File → SplitInBatches → Read Binary File1 → Send Email  
   - Make sure file paths are accessible and permissions are set correctly.  
   - Ensure CSV file has headers `email` and `name` (or adjust expressions accordingly).  
   - Validate SMTP credentials and test sending a single email before bulk run.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                  |
|--------------------------------------------------------------------------------------------------|-------------------------------------------------|
| The workflow requires a pre-formatted CSV with at least two columns: `email` and `name`.         | Input CSV format requirement                      |
| The attachment file names correspond exactly to the `name` field from CSV with `.png` extension. | Naming convention for attachments                 |
| SMTP credentials must be configured prior to execution; use secure OAuth2 if available.           | SMTP setup and security best practices            |
| Allow unauthorized certificates option is enabled for SMTP; disable in production for security.  | SMTP node configuration note                       |
| For large datasets, consider adjusting `SplitInBatches` size to avoid server overload.            | Performance optimization tip                        |

---

This document provides a full structured and detailed reference for the "Bulk Emails with personalized attachment" workflow, enabling reproduction, modification, and troubleshooting.
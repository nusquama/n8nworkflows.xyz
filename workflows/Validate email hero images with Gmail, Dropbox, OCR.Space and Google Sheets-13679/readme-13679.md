Validate email hero images with Gmail, Dropbox, OCR.Space and Google Sheets

https://n8nworkflows.xyz/workflows/validate-email-hero-images-with-gmail--dropbox--ocr-space-and-google-sheets-13679


# Validate email hero images with Gmail, Dropbox, OCR.Space and Google Sheets

# Workflow Reference: Validate Email Hero Images with Gmail, Dropbox, OCR.Space and Google Sheets

This workflow automates the Quality Assurance (QA) process for email marketing assets. It compares the text content of a "Hero Image" found in a live email against a reference "Expected Image" stored in Dropbox. By leveraging Optical Character Recognition (OCR), it detects discrepancies between design specifications and the final email build, logging the results automatically to a Google Sheet.

---

### 1. Workflow Overview

The logic is divided into four functional stages:
- **1.1 Email Acquisition & Extraction:** Retrieves unread emails via Gmail and uses JavaScript to parse the HTML to find the Hero Image URL.
- **1.2 Asset Downloading:** Parallel processes fetch the binary image data from both the live email source (HTML) and the reference source (Dropbox).
- **1.3 OCR Processing:** Sends both images to the OCR.Space API to convert visual text into machine-readable strings.
- **1.4 Comparison & Logging:** Normalizes the extracted text, identifies the match status (Exact, Partial, or Mismatch), and records the findings in Google Sheets.

---

### 2. Block-by-Block Analysis

#### 2.1 Input & Extraction
*   **Nodes Involved:** `When clicking ‘Execute workflow’`, `Get many messages`, `Extract Hero Image SRC From HTML`.
*   **Overview:** Triggers the flow, fetches target emails, and identifies the specific image to be validated.
*   **Node Details:**
    *   **Gmail (Get many messages):** Filters for unread messages from a specific sender (`user@example.com`).
    *   **Code (Extract Hero Image SRC From HTML):** Uses Regex to find `<img>` tags within specific containers (IDs like `heroImage` or `fullWidthImageModule`). It outputs the `src`, `alt`, and `title` attributes along with basic validation flags.
    *   **Edge Cases:** Failure if the email structure doesn't match the expected `id` or `class` patterns defined in the regex.

#### 2.2 Image Retrieval (Binary Conversion)
*   **Nodes Involved:** `Convert Image File from Dropbox to Binary`, `Convert Image File from HTML to Binary`.
*   **Overview:** Converts remote URLs into binary files that can be processed by OCR.
*   **Node Details:**
    *   **HTTP Request:** Performs a GET request to the image URLs.
    *   **Configuration:** `Response Format` is set to `File` to ensure n8n handles the data as a binary object rather than text/JSON.
    *   **Potential Failures:** 404 errors if the Dropbox link is expired or the HTML image path is broken.

#### 2.3 OCR Text Extraction
*   **Nodes Involved:** `Extracting Expected Image Content from OCR1`, `Extracting Actual Image Content from OCR`.
*   **Overview:** Communicates with the OCR.Space API to read text from the binary images.
*   **Node Details:**
    *   **HTTP Request (POST):** Sends `multipart/form-data` containing the binary image.
    *   **Parameters:** Uses an API Key (provided via header) and specifies the language (defaulting to English).
    *   **Requirements:** Requires an active OCR.Space API key.

#### 2.4 Data Consolidation & Reporting
*   **Nodes Involved:** `Combine Image Outputs`, `Extract Content Match Results`, `Log Image Checks to Excel`.
*   **Overview:** Merges the two OCR streams, compares the text, and saves the final report.
*   **Node Details:**
    *   **Merge (Combine Image Outputs):** Synchronizes the parallel paths to ensure both the "Expected" and "Actual" text are available for the next step.
    *   **Code (Extract Content Match Results):** Normalizes strings (lowercase, removes extra spaces) and categorizes the result: *Exact Match*, *Partial Match*, *Mismatch*, or *Missing*.
    *   **Google Sheets (Log Image Checks to Excel):** Appends a row to a specified spreadsheet with the comparison result and the raw extracted text from both sources.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| When clicking ‘Execute workflow’ | Manual Trigger | Manual Start | None | Get many messages | |
| Get many messages | Gmail | Fetch Emails | Manual Trigger | Extract Hero Image SRC From HTML | |
| Extract Hero Image SRC From HTML | Code | HTML Parsing | Get many messages | Convert Image File (Both) | |
| Convert Image File from Dropbox to Binary | HTTP Request | Download Reference | Extract Hero Image | Extracting Expected Content... | Image Validation: Fetch expected image from Dropbox... |
| Convert Image File from HTML to Binary | HTTP Request | Download Live Image | Extract Hero Image | Extracting Actual Content... | Image Validation: Fetch expected image from Dropbox... |
| Extracting Expected Image Content from OCR1 | HTTP Request | OCR (Reference) | Convert Image (Dropbox) | Combine Image Outputs | Image Validation: Fetch expected image from Dropbox... |
| Extracting Actual Image Content from OCR | HTTP Request | OCR (Live) | Convert Image (HTML) | Combine Image Outputs | Image Validation: Fetch expected image from Dropbox... |
| Combine Image Outputs | Merge | Data Sync | OCR Nodes (Both) | Extract Content Match Results | Image Validation: Fetch expected image from Dropbox... |
| Extract Content Match Results | Code | Logic/Comparison | Combine Image Outputs | Log Image Checks to Excel | Image Validation: Fetch expected image from Dropbox... |
| Log Image Checks to Excel | Google Sheets | Data Export | Extract Content Match | None | Image Validation: Fetch expected image from Dropbox... |

---

### 4. Reproducing the Workflow from Scratch

1.  **Trigger:** Add a **Manual Trigger** node.
2.  **Email Source:** Add a **Gmail Node**, set the operation to `Get Many`, and apply a filter for `unread` messages.
3.  **Parsing Logic:** Add a **Code Node**. Use Javascript/Regex to look for the `src` attribute of an `<img>` tag inside a `div` with `id="heroImage"`.
4.  **Download Assets:** 
    *   Add two **HTTP Request Nodes**.
    *   Set **Node A** URL to your Dropbox reference link. 
    *   Set **Node B** URL to the `src` extracted from the Gmail Code node.
    *   **Crucial:** Set both nodes' `Response Format` to `File`.
5.  **OCR Setup:**
    *   Add two more **HTTP Request Nodes** (POST).
    *   URL: `https://api.ocr.space/parse/image`.
    *   Body Content Type: `form-data`.
    *   Parameters: Map the binary file from the previous step to the `file` field.
    *   Headers: Add `apikey` with your OCR.Space key.
6.  **Join Streams:** Add a **Merge Node** to wait for both OCR requests to finish.
7.  **Comparison Logic:** Add a **Code Node**. Write a script to compare `input1.json.ParsedText` with `input2.json.ParsedText`. Output a JSON object with a `result` field (e.g., "Match").
8.  **Logging:** Add a **Google Sheets Node**. Set the operation to `Append`. Map the results from the comparison node to your sheet columns.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **OCR.Space API** | Requires a free or paid API key from [ocr.space](https://ocr.space/ocrapi). |
| **Dropbox Links** | Ensure Dropbox links use `dl.dropboxusercontent.com` or end in `?dl=1` for direct binary access. |
| **HTML Selectors** | The extraction logic expects Marketo-style tags (`mktoImg`, `mktoModule`). Adjust Regex if using different ESPs. |
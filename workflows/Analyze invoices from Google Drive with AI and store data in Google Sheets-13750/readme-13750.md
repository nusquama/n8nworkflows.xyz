Analyze invoices from Google Drive with AI and store data in Google Sheets

https://n8nworkflows.xyz/workflows/analyze-invoices-from-google-drive-with-ai-and-store-data-in-google-sheets-13750


# Analyze invoices from Google Drive with AI and store data in Google Sheets

# Analyze Invoices from Google Drive with AI and Store Data in Google Sheets

### 1. Workflow Overview
This workflow automates the extraction of financial data from invoices stored in Google Drive and centralizes it into a Google Sheets spreadsheet. It is designed to handle multiple file formats, specifically PDFs and images (PNG/JPG), using advanced AI models (GPT-4o and Gemini) to "read" the documents.

**The logic is organized into three main functional blocks:**
*   **1.1 Input Reception & Triggering:** Monitors a specific Google Drive folder for new files or receives images via a Telegram bot to save them into the Drive folder.
*   **1.2 Metadata Mapping & Routing:** Extracts basic file information (MIME type, ID, Owner) and routes the file to the correct processing path based on its extension.
*   **1.3 AI Analysis & Data Storage:** Downloads the file, utilizes AI to extract structured JSON data (Issuer, Date, Total, Tax, etc.), cleans the AI response, and appends the final result to a Google Sheet.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Triggering
This block serves as the entry point. It supports both automated polling of a folder and manual submission via Telegram.

*   **Nodes Involved:** `Telegram Trigger`, `Save_drive`, `Google Drive Trigger`.
*   **Node Details:**
    *   **Telegram Trigger:** Monitors a Telegram bot for incoming messages. Configured to download files/photos directly.
    *   **Save_drive (Google Drive):** Uploads the file received from Telegram to the designated "Facturas" folder. It renames the file using the current date and a unique ID.
    *   **Google Drive Trigger:** Polls a specific folder every minute for new file creations. This is the primary trigger for the analysis pipeline.

#### 2.2 Metadata Mapping & Routing
This block prepares the data for the loop and ensures different file types are handled by the appropriate AI model.

*   **Nodes Involved:** `Mapeo`, `Loop Over Items`, `Switch`.
*   **Node Details:**
    *   **Mapeo (Edit Fields):** Extracts specific metadata from the Google Drive event (File Name, URL, Owner Email, Size, and MimeType) and formats a custom "Date and time" string.
    *   **Loop Over Items (Split in Batches):** Ensures that if multiple files are detected at once, they are processed sequentially to avoid rate limits or overlapping writes.
    *   **Switch:** Routes the workflow based on the `fileExtension` property. It has three outputs: "Imagen png", "PDF", and "Imagen jpg".

#### 2.3 AI Analysis & Data Storage
The core logic where unstructured documents are converted into structured data.

*   **Nodes Involved:** `download image`/`Download PDF`, `analyze image`/`analyze document`, `Edit Fields`/`Edit Fields1`/`Edit Fields2`, `Put data in table`, `Wait`.
*   **Node Details:**
    *   **Download Nodes (Google Drive):** Downloads the binary content of the file using the `File ID` passed from the trigger.
    *   **Analyze Nodes (OpenAI / Gemini):** 
        *   Uses **GPT-4o** for images and **Gemini 2.5 Flash** for PDF documents.
        *   **Prompt:** A strict instruction set requiring a JSON response with fields: *Fecha, Categoría, Emisor, Descripción, Precio (sin IVA), IVA, Total, Moneda, Notas, and Nº de contacto*. It specifically requests a comma as a decimal separator.
    *   **Edit Fields (JSON Cleaning):** Contains a sophisticated JavaScript expression to clean the AI's response. It removes Markdown code fences (```json), handles escaped characters, and ensures the output is a valid JSON array even if the AI adds conversational noise.
    *   **Put data in table (Google Sheets):** Appends the cleaned fields into a specific Google Sheet.
    *   **Wait:** Pauses for 7 seconds after each row is added. This is a safety measure to prevent Google Sheets API concurrency errors during batch processing.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Telegram Trigger | Telegram Trigger | Manual Input | - | Save_drive | |
| Save_drive | Google Drive | File Upload | Telegram Trigger | - | |
| Google Drive Trigger | Google Drive Trigger | Main Trigger | - | Mapeo | Google Drive file detection |
| Mapeo | Edit Fields | Metadata Prep | Google Drive Trigger | Loop Over Items | Google Drive file detection |
| Loop Over Items | Split in Batches | Batch Control | Mapeo, Wait | Switch | |
| Switch | Switch | Format Routing | Loop Over Items | download image, Download PDF, Download image | File type classification |
| download image | Google Drive | Fetch Binary | Switch | analyze image | Invoice analysis (Images/PDF) |
| Download PDF | Google Drive | Fetch Binary | Switch | analyze document | Invoice analysis (Images/PDF) |
| Download image | Google Drive | Fetch Binary | Switch | analyze image1 | Invoice analysis (Images/PDF) |
| analyze image | OpenAI | Image OCR/AI | download image | Edit Fields1 | Invoice analysis (Images/PDF) |
| analyze document | Google Gemini | PDF OCR/AI | Download PDF | Edit Fields | Invoice analysis (Images/PDF) |
| analyze image1 | OpenAI | Image OCR/AI | Download image | Edit Fields2 | Invoice analysis (Images/PDF) |
| Edit Fields1 | Edit Fields | JSON Cleanup | analyze image | Put data in table1 | |
| Edit Fields | Edit Fields | JSON Cleanup | analyze document | Put data in table | |
| Edit Fields2 | Edit Fields | JSON Cleanup | analyze image1 | Put data in table2 | |
| Put data in table1 | Google Sheets | Data Storage | Edit Fields1 | Wait | |
| Put data in table | Google Sheets | Data Storage | Edit Fields | Wait | |
| Put data in table2 | Google Sheets | Data Storage | Edit Fields2 | Wait | |
| Wait | Wait | Delay | Put data in table(s) | Loop Over Items | |

---

### 4. Reproducing the Workflow from Scratch

1.  **Preparation:** Create a Google Drive folder for invoices and a Google Sheet with headers: `Fecha`, `Categoría`, `Emisor`, `Descripción`, `Precio (sin IVA)`, `IVA`, `Total`, `Moneda`, `Notas`, `Nº contacto`, `URL`.
2.  **Trigger Setup:** Add a **Google Drive Trigger** node. Set the event to "File Created" and select your specific folder.
3.  **Metadata Mapping:** Add an **Edit Fields** node (`Mapeo`). Map the incoming JSON properties (`name`, `id`, `mimeType`, etc.) to clean variable names.
4.  **Batching:** Add a **Loop Over Items** node. This ensures the workflow handles folder uploads containing multiple files without crashing.
5.  **Routing:** Add a **Switch** node. Create three rules based on the `fileExtension` string (equals `pdf`, `png`, or `jpg`).
6.  **File Download:** Connect three **Google Drive** nodes (one for each switch path). Set the action to "Download" using the `File ID` from step 3.
7.  **AI Integration:**
    *   For **Images (PNG/JPG):** Add **OpenAI** nodes. Use Model `gpt-4o`. Set resource to "Image" and input type to "Base64".
    *   For **PDFs:** Add a **Google Gemini** node. Use Model `gemini-1.5-flash` (or newer). Set resource to "Document" and input type to "Binary".
    *   *Prompting:* In all AI nodes, use a prompt that demands a specific JSON schema (see Block 2.3 for the prompt content).
8.  **Data Cleaning:** Add **Edit Fields** nodes after each AI node. Use a JavaScript expression to parse the string result: `JSON.parse($json.content.replace(/```json|```/g, ""))`.
9.  **Spreadsheet Integration:** Add **Google Sheets** nodes. Set the action to "Append". Map the cleaned AI fields to your spreadsheet columns.
10. **Loop Completion:** Add a **Wait** node (set to 7 seconds) after the Google Sheets node, then connect it back to the **Loop Over Items** node.
11. **Credentials:** Authenticate Google Drive, Google Sheets, OpenAI, and Google Gemini (Palm API).

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Designed for teams/businesses receiving invoices in Drive. | [Context: Purpose] |
| No accounting software required. | [Context: Requirement] |
| Sequential writing ensured by Wait node. | [Context: Reliability] |
| Supports PDF, PNG, and JPG. | [Context: Capabilities] |
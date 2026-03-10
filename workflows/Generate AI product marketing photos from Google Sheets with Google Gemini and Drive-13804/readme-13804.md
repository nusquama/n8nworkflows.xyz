Generate AI product marketing photos from Google Sheets with Google Gemini and Drive

https://n8nworkflows.xyz/workflows/generate-ai-product-marketing-photos-from-google-sheets-with-google-gemini-and-drive-13804


# Generate AI product marketing photos from Google Sheets with Google Gemini and Drive

# AI Product Marketing Photo Generator: Reference Document

This document provides a technical breakdown of the n8n workflow designed to automate the creation of professional product photography. The workflow retrieves product image URLs from Google Sheets, uses AI (Google Gemini) to analyze and generate composite marketing images featuring a human model, and saves the results back to Google Drive and Google Sheets.

---

### 1. Workflow Overview

The workflow streamlines the production of marketing assets by transforming raw product photos into lifestyle shots. It follows a linear progression through four functional phases:

*   **1.1 Input & Asset Retrieval:** Fetches product URLs from a spreadsheet and downloads both the product image and a pre-defined "model" image from Google Drive.
*   **1.2 AI Analysis & Prompt Engineering:** Uses Google Gemini to identify the product and then uses a LLM chain to craft a highly specific image generation prompt.
*   **1.3 Data Consolidation:** Merges binary data (images) and text data (prompts) to prepare for the generation step.
*   **1.4 AI Generation & Persistence:** Generates the final image using Gemini's image editing capabilities, uploads the file to Google Drive, and updates the Google Sheet with the new image link and the prompt used.

---

### 2. Block-by-Block Analysis

#### Phase 1: Input & Load Product Images
**Overview:** This block handles the ingestion of data and the downloading of physical image files required for the AI process.

*   **Nodes Involved:** `When clicking 'Test workflow'`, `Read Image URLs`, `Download Images`, `Download model image`.
*   **Node Details:**
    *   **Read Image URLs (Google Sheets):** Reads a list of URLs from the "Product Images" sheet. Requires a column named `Image-URL`.
    *   **Download Images (HTTP Request):** Performs a GET request on the URL provided by the sheet to retrieve the product's binary data.
    *   **Download model image (Google Drive):** Downloads a static image file (e.g., `Model.png`) from a specific Drive ID to be used as the human reference in the final composite.

#### Phase 2: Analyze Image & Create Prompt
**Overview:** This block identifies what the product is and generates the creative instructions for the AI image generator.

*   **Nodes Involved:** `Analyze an image`, `Product Photography Prompt`, `Google Gemini Chat Model`.
*   **Node Details:**
    *   **Analyze an image (Google Gemini):** Uses a vision-capable model (Gemini Flash Lite) to describe the input product image in fewer than 5 words.
    *   **Product Photography Prompt (AI Chain):** An LLM chain that takes the brief description and follows strict instructions to build a professional photography prompt. It ensures the output specifies the model's visibility, lighting, and "cinematic look."
    *   **Google Gemini Chat Model:** The underlying brain for the prompt generation, configured via Google Gemini (PaLM) API.

#### Phase 3: Combine Product + Model + Prompt
**Overview:** Synchronizes the disparate data streams (Product Image, Model Image, and Prompt text) into a single object per item.

*   **Nodes Involved:** `Merge`, `Merge1`.
*   **Node Details:**
    *   **Merge:** Combines the product image binary data with the generated text prompt.
    *   **Merge1:** Combines the result of the previous merge with the binary data of the human model.
    *   **Logic:** Both nodes use "Combine by Position" to ensure that the $i$-th product corresponds to the generated prompt and the model image.

#### Phase 4: Generate AI Image & Save Results
**Overview:** The final execution phase where the AI "edits" the images together and the results are stored externally.

*   **Nodes Involved:** `Loop Over Items`, `Edit an image`, `Upload to Drive`, `Insert Image URL in Table`.
*   **Node Details:**
    *   **Loop Over Items (Split in Batches):** Processes each product one by one to prevent timeouts and ensure structured output.
    *   **Edit an image (Google Gemini):** Uses the `image:edit` operation. It takes the text prompt and two binary inputs (product and model) to generate the final marketing photo.
    *   **Upload to Drive (Google Drive):** Saves the generated image to a specific folder using a timestamped filename.
    *   **Insert Image URL in Table (Google Sheets):** Appends or updates the "Output Images" sheet with the Google Drive `webViewLink`, the `Prompt`, and matches it to the original `Image-URL`.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Read Image URLs | Google Sheets | Data Retrieval | Manual Trigger | Download Images | Phase 1 – Input & Load Product Images |
| Download Images | HTTP Request | Asset Loading | Read Image URLs | Merge, Download model, Analyze | Phase 1 – Input & Load Product Images |
| Download model image | Google Drive | Asset Loading | Download Images | Merge1 | Phase 1 – Input & Load Product Images |
| Analyze an image | Google Gemini | Visual Analysis | Download Images | Product Photography Prompt | Phase 2 – Analyze Image & Create Prompt |
| Product Photography Prompt | Chain LLM | Prompt Engineering | Analyze an image | Merge | Phase 2 – Analyze Image & Create Prompt |
| Google Gemini Chat Model | Gemini Chat Model | LLM Provider | (N/A) | Product Photography Prompt | Phase 2 – Analyze Image & Create Prompt |
| Merge | Merge | Data Sync | Download Images, Prompt | Merge1 | Phase 3 – Combine Product + Model + Prompt |
| Merge1 | Merge | Data Sync | Download model, Merge | Loop Over Items | Phase 3 – Combine Product + Model + Prompt |
| Loop Over Items | Split in Batches | Flow Control | Merge1 | Edit an image | Phase 4 – Generate AI Image & Save Results |
| Edit an image | Google Gemini | Image Generation | Loop Over Items | Upload to Drive | Phase 4 – Generate AI Image & Save Results |
| Upload to Drive | Google Drive | Storage | Edit an image | Insert Image URL | Phase 4 – Generate AI Image & Save Results |
| Insert Image URL in Table | Google Sheets | Reporting | Upload to Drive | Loop Over Items | Phase 4 – Generate AI Image & Save Results |

---

### 4. Reproducing the Workflow from Scratch

1.  **Preparation:**
    *   Create a Google Sheet with two tabs: "Product Images" (Column: `Image-URL`) and "Output Images" (Columns: `Image-URL`, `Prompt`, `Output`).
    *   Upload a "Model" image to Google Drive to serve as your consistent brand face.
2.  **Trigger & Input:**
    *   Add a **Manual Trigger**.
    *   Add a **Google Sheets Node** (Read Image URLs). Configure it to "Get Many" rows from your source sheet.
    *   Add an **HTTP Request Node**. Set the URL to `{{ $json['Image-URL'] }}` to download the product image.
3.  **Asset & Analysis:**
    *   Add a **Google Drive Node** (Download model image). Use the "Download" operation with the File ID of your model photo.
    *   Add a **Google Gemini Node** (Analyze an image). Set operation to "Image: Analyze" and input the binary from the HTTP node.
4.  **Prompt Generation:**
    *   Add a **Basic LLM Chain** node. Connect a **Google Gemini Chat Model** node to its AI Model input.
    *   In the Chain node, provide the instructions to create a marketing prompt based on the analysis output.
5.  **Data Merging:**
    *   Add a **Merge Node** to join the HTTP Request (Product Image) and the LLM Chain (Prompt).
    *   Add a second **Merge Node** to join the first Merge with the Google Drive Download (Model Image). Set both to "Combine by Position."
6.  **Looping & Generation:**
    *   Add a **Split in Batches** node to process items sequentially.
    *   Add a **Google Gemini Node** (Edit an image). Configure it to take the Prompt and two binary properties (product and model).
7.  **Finalization:**
    *   Add a **Google Drive Node** to upload the output. Use the "Upload" operation.
    *   Add a **Google Sheets Node** (Append or Update). Match on `Image-URL` and map the Drive `webViewLink` to the `Output` column.
    *   Connect the final Google Sheets node back to the **Split in Batches** node to continue the loop.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Use an input sheet with `Image-URL` and an output sheet with `Image-URL`, `Prompt`, and `Output`. | Setup Requirement |
| Workflow uses `gemini-2.5-flash-lite` for analysis and `gemini-3-pro-image-preview` for generation. | Model Versions |
| The logic ensures that if a product is wearable, the AI is instructed to place it ON the model. | Creative Logic |
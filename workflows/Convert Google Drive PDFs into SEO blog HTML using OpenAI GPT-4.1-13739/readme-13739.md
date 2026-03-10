Convert Google Drive PDFs into SEO blog HTML using OpenAI GPT-4.1

https://n8nworkflows.xyz/workflows/convert-google-drive-pdfs-into-seo-blog-html-using-openai-gpt-4-1-13739


# Convert Google Drive PDFs into SEO blog HTML using OpenAI GPT-4.1

# Reference Documentation: Convert Google Drive PDFs into SEO blog HTML using OpenAI GPT-4.1

## 1. Workflow Overview
This workflow automates the transformation of financial or market-related PDF documents into high-quality, SEO-optimized blog posts in HTML format. It is designed to streamline content creation by fetching source documents from a specific Google Drive folder, processing them through an AI model (GPT-4.1), and saving the formatted output back to a designated folder.

The logic is organized into three primary functional blocks:
*   **1.1 Input & Retrieval:** Scanning a source folder and managing the iteration of discovered files.
*   **1.2 Content Extraction:** Downloading binary PDF data and converting it into machine-readable text.
*   **1.3 AI Synthesis & Storage:** Applying complex SEO instructions via OpenAI to generate HTML content and saving the result as a new file.

---

## 2. Block-by-Block Analysis

### 2.1 Input & Retrieval
**Overview:** This block triggers the process and identifies all PDF files within a specific Google Drive directory, preparing them for sequential processing.

*   **Nodes Involved:** `When clicking ‘Execute workflow’`, `Search files and folders`, `Loop Over Items`.
*   **Node Details:**
    *   **When clicking ‘Execute workflow’:** Manual trigger to start the session.
    *   **Search files and folders (Google Drive):** Filters for files within a specific Folder ID (`Input-PDFs`). It is configured to return all items to ensure no data is missed.
        *   *Edge Cases:* Empty folder results in no downstream execution; Folder ID permissions must be shared with the OAuth2 account.
    *   **Loop Over Items (Split in Batches):** Ensures the workflow processes one file at a time. This prevents API rate limiting on the OpenAI side and memory exhaustion during large PDF extractions.

### 2.2 Content Extraction
**Overview:** This block handles the binary data of the files, converting the visual/document format of a PDF into a string of text that the AI can interpret.

*   **Nodes Involved:** `Download file`, `Extract from File`.
*   **Node Details:**
    *   **Download file (Google Drive):** Uses the `File ID` passed from the search node to fetch the actual binary content of the PDF.
        *   *Potential Failure:* File deletion between the search and download steps.
    *   **Extract from File:** Specifically configured for PDF operation. It parses the binary data and outputs the raw text into the `$json.text` variable.
        *   *Technical Role:* Text extraction (OCR is not explicitly enabled, so it relies on text-layer PDFs).

### 2.3 AI Synthesis & Storage
**Overview:** The core logic where the raw text is transformed into a structured blog post based on strict SEO guidelines, followed by the generation of the final HTML file.

*   **Nodes Involved:** `Message a model`, `Create file from text`.
*   **Node Details:**
    *   **Message a model (OpenAI):** Uses the `GPT-4.1` model. It contains a comprehensive system prompt requiring:
        *   Minimum 1200 words, H1/H2/H3 structure, and bullet points.
        *   Specific tone (Professional Finance/Markets - AngelOne style).
        *   SEO metadata (150-160 character description).
        *   Output format: Strict HTML.
    *   **Create file from text (Google Drive):** Takes the AI output and creates a new `.html` file. 
        *   *Key Expression:* Uses a JavaScript snippet `{{ $('Search files and folders').item.json.name.split('.').slice(0, -1).join('.') + '.html' }}` to rename the file from `.pdf` to `.html` while keeping the original prefix.
        *   *Destination:* Saves to the `Generated-Blogs` folder.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| When clicking ‘Execute workflow’ | Manual Trigger | Entry Point | None | Search files... | |
| Search files and folders | Google Drive | Discovery | Manual Trigger | Loop Over Items | PDF retrieval and sequential processing |
| Loop Over Items | Split in Batches | Flow Control | Search files... | Download file | PDF retrieval and sequential processing |
| Download file | Google Drive | Data Fetch | Loop Over Items | Extract from File | PDF retrieval and sequential processing |
| Extract from File | Extract from File | Text Parsing | Download file | Message a model | PDF retrieval and sequential processing |
| Message a model | OpenAI | AI Content Gen | Extract from File | Create file... | HTML file creation |
| Create file from text | Google Drive | File Storage | Message a model | Loop Over Items | HTML file creation |

---

## 4. Reproducing the Workflow from Scratch

1.  **Preparation:** Create two folders in Google Drive: `Input-PDFs` and `Generated-Blogs`.
2.  **Trigger:** Add a **Manual Trigger** node.
3.  **Search Step:** Add a **Google Drive** node, set Action to `Search`. Select the `Input-PDFs` folder. Ensure "Return All" is toggled ON.
4.  **Batching:** Add a **Loop Over Items** node. Connect the Search node to this.
5.  **Download Step:** Add a **Google Drive** node, set Action to `Download`. Use the expression `{{ $json.id }}` to target the file from the loop.
6.  **Extraction:** Add an **Extract from File** node. Set the Operation to `PDF`. This will turn the binary download into a `text` property.
7.  **AI Integration:** Add an **OpenAI** node (Message a Model).
    *   **Model:** Select `gpt-4.1`.
    *   **Prompt:** Input the SEO requirements (1200 words, HTML format, Meta Description, financial tone). Reference the text using `{{ $json.text }}`.
8.  **File Creation:** Add a **Google Drive** node, set Action to `Create from Text`.
    *   **Content:** `{{ $json.output[0].content[0].text }}`.
    *   **File Name:** `{{ $node["Search files and folders"].json["name"].replace(".pdf", ".html") }}`.
    *   **Location:** Select the `Generated-Blogs` folder.
9.  **Closing the Loop:** Connect the output of the "Create file from text" node back to the input of the **Loop Over Items** node to process the next file.
10. **Credentials:** Authenticate your Google Drive (OAuth2) and OpenAI (API Key) accounts in their respective nodes.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Full Workflow Overview** | [Link to Source Documentation/Template if applicable] |
| **SEO Prompting** | The prompt is tailored for financial markets (AngelOne). Adjust the "Tone" section for different industries. |
| **Efficiency** | Using "Split in Batches" is critical here to avoid OpenAI 429 errors (Too Many Requests) when processing multiple large PDFs. |
| **File Naming** | The workflow uses a split/slice method to ensure that filenames with multiple dots (e.g., `report.v1.pdf`) are correctly renamed to `.html`. |
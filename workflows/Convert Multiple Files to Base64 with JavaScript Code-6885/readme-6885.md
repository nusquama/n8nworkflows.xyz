Convert Multiple Files to Base64 with JavaScript Code

https://n8nworkflows.xyz/workflows/convert-multiple-files-to-base64-with-javascript-code-6885


# Convert Multiple Files to Base64 with JavaScript Code

### 1. Workflow Overview

This workflow is designed to automate the process of converting multiple binary files into Base64-encoded strings within n8n. It targets use cases where an API or downstream system requires files in Base64 format, especially when handling batches of files that standard extraction nodes cannot process efficiently.

The workflow is logically organized into the following blocks:

- **1.1 Input Reception:** Manual trigger to initiate the workflow execution.
- **1.2 File Acquisition and Extraction:** Download a ZIP file and unzip it to extract multiple binary files.
- **1.3 Base64 Encoding Processing:** Use a Code node with custom JavaScript to convert each extracted binary file into a Base64 string.
- **1.4 Documentation and Guidance:** A sticky note providing usage instructions and contextual information.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block provides a manual trigger to start the workflow, allowing users to execute the process on demand.

**Nodes Involved:**  
- When clicking ‘Execute workflow’

**Node Details:**  

- **Node Name:** When clicking ‘Execute workflow’  
- **Type:** Manual Trigger  
- **Technical Role:** Acts as the entry point for the workflow, enabling manual execution.  
- **Configuration:** No parameters configured; default manual trigger behavior.  
- **Expressions/Variables:** None.  
- **Input Connections:** None (start node).  
- **Output Connections:** Connects to "Download n8n demo website zip" node.  
- **Version-Specific Requirements:** Compatible with n8n v1+.  
- **Potential Failure Types:** None (manual trigger).  
- **Sub-workflow Reference:** None.

---

#### 1.2 File Acquisition and Extraction

**Overview:**  
This block downloads a ZIP archive from a remote repository and extracts its contents to prepare multiple binary files for encoding.

**Nodes Involved:**  
- Download n8n demo website zip  
- Unzip

**Node Details:**  

- **Node Name:** Download n8n demo website zip  
- **Type:** HTTP Request  
- **Technical Role:** Downloads a ZIP file from a specified URL.  
- **Configuration:**  
  - URL: `https://github.com/n8n-io/n8n-demo-website/archive/refs/heads/main.zip`  
  - Response format set to “file” to handle binary download correctly.  
- **Expressions/Variables:** None.  
- **Input Connections:** Receives trigger from "When clicking ‘Execute workflow’".  
- **Output Connections:** Passes output binary data to "Unzip" node.  
- **Version-Specific Requirements:** Uses HTTP Request node v4.2 or higher to support response as file.  
- **Potential Failure Types:**  
  - Network errors or download failure (e.g., 404 if URL changes).  
  - Timeout or slow response.  
  - Incorrect file format or corrupted download.  
- **Sub-workflow Reference:** None.

- **Node Name:** Unzip  
- **Type:** Compression  
- **Technical Role:** Extracts files from the downloaded ZIP archive to multiple binary file items.  
- **Configuration:** Default unzip settings (no additional parameters).  
- **Expressions/Variables:** None.  
- **Input Connections:** Receives binary ZIP file from "Download n8n demo website zip".  
- **Output Connections:** Sends extracted binary files to "Encode to base64" node.  
- **Version-Specific Requirements:** Node version 1.1 or higher recommended for stable unzip support.  
- **Potential Failure Types:**  
  - Invalid or corrupted ZIP file errors.  
  - Large ZIP files causing memory or timeout issues.  
- **Sub-workflow Reference:** None.

---

#### 1.3 Base64 Encoding Processing

**Overview:**  
A Code node processes each extracted binary file and converts it into a Base64-encoded string, preserving the file path and name for downstream usage.

**Nodes Involved:**  
- Encode to base64

**Node Details:**  

- **Node Name:** Encode to base64  
- **Type:** Code  
- **Technical Role:** Executes custom JavaScript to loop through multiple binary files, convert each to Base64, and prepare structured output.  
- **Configuration:**  
  - JavaScript code iterates over all binary files in the input item’s binary property.  
  - Uses `this.helpers.getBinaryDataBuffer()` to retrieve the buffer for each file key.  
  - Constructs file paths using directory and filename where available.  
  - Converts buffers to Base64 strings via `Buffer.from().toString('base64')`.  
  - Returns an object containing an array of files with `path` and `data` keys.  
- **Expressions/Variables:**  
  - `$input.first().binary` to access binary data.  
  - Try-catch blocks for error handling per file.  
- **Input Connections:** Receives multiple binary files from "Unzip".  
- **Output Connections:** Outputs an object with a `files` array containing encoded files; can be connected to subsequent nodes for uploading or further processing.  
- **Version-Specific Requirements:** Requires n8n v0.153.0+ for usage of `this.helpers.getBinaryDataBuffer()` and async code in Code node.  
- **Potential Failure Types:**  
  - Missing binary data or undefined keys.  
  - Errors in buffer retrieval or conversion.  
  - Files without directory or filename properties handled gracefully but may cause path inconsistencies.  
- **Sub-workflow Reference:** None.

---

#### 1.4 Documentation and Guidance

**Overview:**  
A sticky note node providing instructions, context, and a link to an external detailed blog post explaining the workflow’s rationale and usage.

**Nodes Involved:**  
- Sticky Note

**Node Details:**  

- **Node Name:** Sticky Note  
- **Type:** Sticky Note  
- **Technical Role:** Provides descriptive information and guidance to users and maintainers about the workflow.  
- **Configuration:**  
  - Content includes workflow purpose, instructions, modification tips, and a hyperlink to a blog post:  
    [https://n8n-tips.blogspot.com/2025/08/from-binary-to-base64-guide-to-file.html](https://n8n-tips.blogspot.com/2025/08/from-binary-to-base64-guide-to-file.html)  
  - Dimensions: width 720, height 384 pixels.  
- **Expressions/Variables:** None.  
- **Input Connections:** None.  
- **Output Connections:** None.  
- **Version-Specific Requirements:** None.  
- **Potential Failure Types:** None.

---

### 3. Summary Table

| Node Name                     | Node Type        | Functional Role                           | Input Node(s)                   | Output Node(s)                 | Sticky Note                                                                                                  |
|-------------------------------|------------------|-----------------------------------------|--------------------------------|-------------------------------|--------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger   | Entry point to start workflow execution | None                           | Download n8n demo website zip  |                                                                                                              |
| Download n8n demo website zip  | HTTP Request     | Downloads ZIP archive containing files  | When clicking ‘Execute workflow’ | Unzip                         |                                                                                                              |
| Unzip                         | Compression      | Extracts multiple binary files from ZIP | Download n8n demo website zip  | Encode to base64               |                                                                                                              |
| Encode to base64              | Code             | Converts binary files to Base64 strings | Unzip                         | None                          |                                                                                                              |
| Sticky Note                   | Sticky Note      | Provides workflow instructions and info | None                           | None                          | ## Base64 Encode Multiple Binary Files with a Code Node This template demonstrates how to handle multiple binary files in n8n by using a Code node to convert them into a Base64 encoded string. It's particularly useful when an API requires file uploads in this format and the standard 'Extract From File' node is not sufficient for batch processing. The workflow starts by downloading a ZIP file, unzipping it to get multiple binary files, and then uses a Code node with custom JavaScript to encode each file individually. Instructions 1. Download and import this template into your n8n instance. 2. Run the workflow once to see how it downloads, unzips, and then encodes multiple files. 3. Modify the 'HTTP Request' node to download your own binary file or a ZIP file containing multiple files. 4. Update the 'Code' node if you need to adjust the output format or file paths. 5. Use the output of the 'Code' node in a subsequent node, such as another 'HTTP Request' to send the Base64-encoded files to your desired API. A link to the full blog post is available [here](https://n8n-tips.blogspot.com/2025/08/from-binary-to-base64-guide-to-file.html) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node:**  
   - Node Type: Manual Trigger  
   - Name: "When clicking ‘Execute workflow’"  
   - No parameters needed. This node will start the workflow manually.

2. **Add an HTTP Request Node:**  
   - Node Type: HTTP Request  
   - Name: "Download n8n demo website zip"  
   - Set Method to `GET`.  
   - Set URL to: `https://github.com/n8n-io/n8n-demo-website/archive/refs/heads/main.zip`  
   - Under Options → Response → Response Format, select `File` to ensure binary download.  
   - Connect its input to the Manual Trigger node.

3. **Add a Compression Node:**  
   - Node Type: Compression  
   - Name: "Unzip"  
   - Operation: Default (Unzip)  
   - Connect its input to the output of "Download n8n demo website zip".  
   - This node extracts the ZIP archive into multiple binary files.

4. **Add a Code Node:**  
   - Node Type: Code  
   - Name: "Encode to base64"  
   - Set Language to `JavaScript`.  
   - Paste the following JavaScript code:

```javascript
const results = [];

for (const file in $input.first().binary) {
  try {
    const bin = $input.first().binary[file];
    const binBuffer = await this.helpers.getBinaryDataBuffer(0, file);

    const path = bin.directory ? `${bin.directory}/${bin.fileName}` : bin.fileName;

    results.push({
      path: path,
      data: Buffer.from(binBuffer).toString('base64'),
    });
  } catch (error) {
    console.error(`Error processing file "${file}": ${error.message}`);
  }
}

return { files: results };
```

   - Connect its input to the output of "Unzip".  
   - This node converts each binary file to a Base64 string, preserving its path.

5. **Add a Sticky Note Node (Optional but Recommended):**  
   - Node Type: Sticky Note  
   - Name: "Sticky Note"  
   - Content:

```
## Base64 Encode Multiple Binary Files with a Code Node
This template demonstrates how to handle multiple binary files in n8n by using a Code node to convert them into a Base64 encoded string. It's particularly useful when an API requires file uploads in this format and the standard 'Extract From File' node is not sufficient for batch processing. The workflow starts by downloading a ZIP file, unzipping it to get multiple binary files, and then uses a Code node with custom JavaScript to encode each file individually.
### Instructions
1. Download and import this template into your n8n instance.
2. Run the workflow once to see how it downloads, unzips, and then encodes multiple files.
3. Modify the 'HTTP Request' node to download your own binary file or a ZIP file containing multiple files.
4. Update the 'Code' node if you need to adjust the output format or file paths.
5. Use the output of the 'Code' node in a subsequent node, such as another 'HTTP Request' to send the Base64-encoded files to your desired API.

A link to the full blog post is available [here](https://n8n-tips.blogspot.com/2025/08/from-binary-to-base64-guide-to-file.html)
```

6. **Verify Connections and Execute:**  
   - Connect nodes in this order:  
     Manual Trigger → HTTP Request → Compression → Code  
   - The Sticky Note is unconnected but placed for reference.

7. **Credentials:**  
   - No external credentials are required for this workflow as HTTP Request accesses a public URL.

8. **Testing and Usage:**  
   - Execute the Manual Trigger node to run the workflow.  
   - Inspect the output of the "Encode to base64" node to verify the Base64-encoded files array.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| This workflow is a practical example for batch processing multiple binary files into Base64 within n8n using a Code node. It addresses limitations of standard extraction nodes when dealing with multiple files. | Workflow purpose and use case context. |
| The included blog post provides an extended explanation and additional use cases: [From Binary to Base64: Guide to File Handling in n8n](https://n8n-tips.blogspot.com/2025/08/from-binary-to-base64-guide-to-file.html) | External resource for deeper understanding. |

---

**Disclaimer:** The provided content is exclusively derived from an automated workflow created with n8n, complying strictly with current content policies and containing no illegal or protected elements. All manipulated data is legal and publicly accessible.
Convert Multiple Binary Files to Base64 JSON Arrays with No Custom Code

https://n8nworkflows.xyz/workflows/convert-multiple-binary-files-to-base64-json-arrays-with-no-custom-code-9481


# Convert Multiple Binary Files to Base64 JSON Arrays with No Custom Code

### 1. Workflow Overview

This n8n workflow provides a no-code, reusable solution for converting multiple binary files contained within a single item (typically from an unzipped archive) into a JSON array of Base64-encoded file data along with their respective file paths. It targets use cases where APIs or integrations require batch file uploads or complex data structures formatted as JSON arrays, each element comprising a file path and its Base64-encoded content.

**Logical blocks:**

- **1.1 Input Reception**  
  Handles manual triggering and downloads a zipped archive containing multiple files.

- **1.2 Archive Extraction**  
  Unzips the downloaded archive to expose individual binary files.

- **1.3 File Isolation and Encoding**  
  Separates each binary file into individual items and converts each file’s binary content into Base64 strings.

- **1.4 File Metadata Reconstruction**  
  Builds the full file path for each file and prepares the data structure.

- **1.5 Aggregation of Output**  
  Combines all individual encoded files into a single JSON array for downstream use.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block initiates the workflow manually, then downloads a zipped archive containing multiple binary files.

- **Nodes Involved:**  
  - Start  
  - Download Demo Website

- **Node Details:**

  - **Start**  
    - Type: Manual Trigger  
    - Role: Entry point for manual initiation of the workflow  
    - Configuration: Default, no parameters  
    - Inputs: None  
    - Outputs: Connected to "Download Demo Website"  
    - Failure cases: None typical; user-trigger dependent

  - **Download Demo Website**  
    - Type: HTTP Request  
    - Role: Downloads a zipped archive file from a fixed URL (`https://github.com/n8n-io/n8n-demo-website/archive/refs/heads/main.zip`)  
    - Configuration:  
      - URL set as above  
      - Response format set to "File" to receive binary content  
    - Inputs: Receives trigger from Start node  
    - Outputs: Binary file data of the archive passed to "Unzip Demo Website"  
    - Failure cases: Network errors, URL unreachable, file download failure, timeouts

#### 2.2 Archive Extraction

- **Overview:**  
  This block unpacks the downloaded zip file to extract multiple binary files encapsulated inside.

- **Nodes Involved:**  
  - Unzip Demo Website

- **Node Details:**

  - **Unzip Demo Website**  
    - Type: Compression (Unzip)  
    - Role: Extracts individual files from the zipped archive  
    - Configuration: Default unzip settings (no special parameters)  
    - Inputs: Receives binary zip file from "Download Demo Website"  
    - Outputs: Outputs an item containing multiple binary files in a single item (each file as a key in the binary property)  
    - Failure cases: Corrupt archive, unsupported compression format, extraction failure

#### 2.3 File Isolation and Encoding

- **Overview:**  
  Separates each binary file into individual items and converts each binary file into a Base64-encoded string.

- **Nodes Involved:**  
  - Split Out Files  
  - Encode Files to Base64

- **Node Details:**

  - **Split Out Files**  
    - Type: Split Out  
    - Role: Splits the single item containing multiple binary files into multiple items, each holding one binary file  
    - Configuration:  
      - Field to split out: `$binary` (all binary keys/files)  
    - Inputs: Receives zipped file contents from "Unzip Demo Website"  
    - Outputs: Multiple items, each with one binary file  
    - Failure cases: No binary files present, empty input

  - **Encode Files to Base64**  
    - Type: Extract From File  
    - Role: Converts the binary content of each isolated file into a Base64 string  
    - Configuration:  
      - Operation: `binaryToProperty`  
      - Binary property name selected dynamically using expression: `{{ $binary.keys()[0] }}` to target the correct binary data key  
      - Option enabled to keep original binary data intact  
    - Inputs: Single-item binary files from "Split Out Files"  
    - Outputs: Items with added JSON property `data` containing Base64 string  
    - Failure cases: Missing or malformed binary data, expression evaluation errors

#### 2.4 File Metadata Reconstruction

- **Overview:**  
  Adds a reconstructed full file path (including directory if present) to each encoded file item and prepares the data structure for aggregation.

- **Nodes Involved:**  
  - Add Path to Files

- **Node Details:**

  - **Add Path to Files**  
    - Type: Set  
    - Role: Adds two JSON properties to each item:  
      - `path`: Constructed file path combining directory and filename if directory exists  
      - `data`: The Base64-encoded string from previous node  
    - Configuration:  
      - `path` value uses an expression:  
        ```
        {{ $binary[$binary.keys()[0]].directory ? $binary[$binary.keys()[0]].directory + '/' : ''}}{{ $binary[$binary.keys()[0]].fileName }}
        ```  
      - `data` value is set to `{{ $json.data }}`, passing through the encoded string  
    - Inputs: Receives Base64 encoded items from "Encode Files to Base64"  
    - Outputs: Items enriched with `path` and `data` keys, ready for aggregation  
    - Failure cases: Missing binary metadata properties, expression errors

#### 2.5 Aggregation of Output

- **Overview:**  
  Combines all individual file items into one single JSON object containing an array of files, each with its path and Base64 data, matching expected API payload format.

- **Nodes Involved:**  
  - Aggregate Output

- **Node Details:**

  - **Aggregate Output**  
    - Type: Aggregate  
    - Role: Aggregates all incoming items into a single item with a `files` array property  
    - Configuration:  
      - Aggregate mode: `aggregateAllItemData` (collects all item JSON data into an array)  
      - Destination field name: `files`  
    - Inputs: Receives multiple enriched items from "Add Path to Files"  
    - Outputs: Single item with JSON property `files` = array of all file objects  
    - Failure cases: No input items, aggregation errors

---

### 3. Summary Table

| Node Name            | Node Type             | Functional Role                         | Input Node(s)         | Output Node(s)        | Sticky Note                                                                                                   |
|----------------------|-----------------------|---------------------------------------|-----------------------|-----------------------|---------------------------------------------------------------------------------------------------------------|
| Start                | Manual Trigger        | Initiates workflow manually           | —                     | Download Demo Website  |                                                                                                               |
| Download Demo Website | HTTP Request          | Downloads zipped archive file          | Start                 | Unzip Demo Website     |                                                                                                               |
| Unzip Demo Website    | Compression (Unzip)   | Extracts files from zipped archive     | Download Demo Website  | Split Out Files        |                                                                                                               |
| Split Out Files       | Split Out             | Splits multiple binary files into items| Unzip Demo Website     | Encode Files to Base64 |                                                                                                               |
| Encode Files to Base64| Extract From File     | Converts binary files to Base64 strings| Split Out Files        | Add Path to Files      |                                                                                                               |
| Add Path to Files     | Set                   | Adds full file path and data properties| Encode Files to Base64 | Aggregate Output       |                                                                                                               |
| Aggregate Output      | Aggregate             | Combines all files into single JSON array| Add Path to Files       | —                     |                                                                                                               |
| Sticky Note          | Sticky Note           | Documentation and explanation          | —                     | —                     | ## No-Code: Convert Multiple Binary Files to Base64\n\nThis template provides a robust, purely **no-code** solution for a common integration challenge: converting multiple binary files contained within a single n8n item (e.g., after unzipping an archive) into a structured JSON array of Base64 encoded strings.\n\nFor a detailed walkthrough, including the explanation behind the dynamic expressions and why this is superior to the custom code solution, check out the full blog post: [The No-Code Evolution: Base64 Encoding Multiple Files in n8n (Part 2)](https://n8nplaybook.com/post/2025/10/no-code-base64-encoding-in-n8n). |

---

### 4. Reproducing the Workflow from Scratch

1. **Add a Manual Trigger node** named `Start` to initiate the workflow.

2. **Create an HTTP Request node** named `Download Demo Website`:
   - Set **HTTP Method** to `GET`.
   - Set **URL** to `https://github.com/n8n-io/n8n-demo-website/archive/refs/heads/main.zip`.
   - Under **Options**, set **Response Format** > **Response** > **Response Format** to `File`.
   - Connect `Start` node’s output to this node.

3. **Add a Compression node** named `Unzip Demo Website`:
   - Set mode to `Unzip` (default).
   - Connect output of `Download Demo Website` to this node.

4. **Add a Split Out node** named `Split Out Files`:
   - Set **Field To Split Out** to `binary` (use the expression `$binary`).
   - Connect output of `Unzip Demo Website` to this node.

5. **Add an Extract From File node** named `Encode Files to Base64`:
   - Set **Operation** to `binaryToProperty`.
   - For **Binary Property Name**, enter dynamic expression `{{ $binary.keys()[0] }}` to always select the current binary key.
   - Enable **Keep Source** to retain original binary data.
   - Connect output of `Split Out Files` to this node.

6. **Add a Set node** named `Add Path to Files`:
   - Add two fields:
     - `path` (string): Set value to the expression  
       ```
       {{ $binary[$binary.keys()[0]].directory ? $binary[$binary.keys()[0]].directory + '/' : ''}}{{ $binary[$binary.keys()[0]].fileName }}
       ```
       This reconstructs the full file path including the directory if available.
     - `data` (string): Set value to expression `{{ $json.data }}` which holds the Base64-encoded content.
   - Connect output of `Encode Files to Base64` to this node.

7. **Add an Aggregate node** named `Aggregate Output`:
   - Set **Aggregate** mode to `aggregateAllItemData`.
   - Set **Destination Field Name** to `files`.
   - Connect output of `Add Path to Files` to this node.

8. **Optionally, add a Sticky Note node** with workflow explanation and instructions for documentation purposes.

9. **Save and activate the workflow.**

**Credentials:**  
No special credentials are required for this workflow since the HTTP Request accesses a public URL, and all nodes use built-in functionality.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                   | Context or Link                                                                                                               |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------|
| This workflow is a no-code approach to converting multiple binary files within n8n into Base64-encoded JSON arrays, avoiding the need for custom JavaScript code. The dynamic expressions used leverage the binary property keys to handle files flexibly.                                                                       | Sticky Note in workflow; foundational principle of the workflow                                                                |
| For an extended explanation of the dynamic expressions used and the advantages of this no-code approach over custom code solutions, see the blog post: [The No-Code Evolution: Base64 Encoding Multiple Files in n8n (Part 2)](https://n8nplaybook.com/post/2025/10/no-code-base64-encoding-in-n8n)                                | https://n8nplaybook.com/post/2025/10/no-code-base64-encoding-in-n8n                                                           |
| The approach generalizes to any zipped archive source and can be adjusted by replacing the initial HTTP Request with any binary file source node, making it highly versatile for integration scenarios requiring batch file encoding.                                                                                          | General use case note                                                                                                          |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created using n8n, an integration and automation tool. This process strictly adheres to current content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.
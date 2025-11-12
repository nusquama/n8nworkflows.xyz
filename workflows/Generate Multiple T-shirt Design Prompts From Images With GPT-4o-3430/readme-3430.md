Generate Multiple T-shirt Design Prompts From Images With GPT-4o

https://n8nworkflows.xyz/workflows/generate-multiple-t-shirt-design-prompts-from-images-with-gpt-4o-3430


# Generate Multiple T-shirt Design Prompts From Images With GPT-4o

### 1. Workflow Overview

This workflow automates the generation of multiple unique T-shirt design prompts inspired by an original image using GPT-4o AI models. It targets users such as print-on-demand entrepreneurs, logo designers, and digital artists who want to efficiently create stylized AI image generation prompts without manual trial and error.

The workflow is divided into the following logical blocks:

- **1.1 Input Reception**: Monitors a local folder for newly saved images to trigger the workflow.
- **1.2 Image Acquisition & Conversion**: Reads the triggered image file and prepares it for AI analysis.
- **1.3 Image Analysis AI Agent**: Uses GPT-4o to analyze the image deeply, extracting a detailed description and original text phrase.
- **1.4 Prompt/Text Generation AI Agent**: Generates five new image prompts and nine new text phrases based on the analysis.
- **1.5 Output Preparation and Saving**: Converts AI-generated data into text files and saves them to a designated folder for later use.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
This block continuously watches a specified local folder. When a user saves an image file into this folder, the workflow is triggered automatically.

- **Nodes Involved:**  
  - Local File Trigger

- **Node Details:**  
  - **Local File Trigger**  
    - Type: Trigger node to monitor local file system changes.  
    - Configuration: Set to watch a user-specified folder path on the local machine (user sets this path).  
    - Key expressions/variables: None by default, but path must be set by the user before running.  
    - Input: None (trigger node).  
    - Output: Emits event data containing the file path and metadata of the newly saved image.  
    - Edge cases: Folder path misconfiguration will prevent triggering; saving unsupported file types may cause downstream errors.  
    - Version: Compatible with n8n v1+.

#### 2.2 Image Acquisition & Conversion

- **Overview:**  
After the trigger, this block reads the saved image file into workflow memory and converts it into a format suitable for AI analysis.

- **Nodes Involved:**  
  - Converter (Code node)  
  - Get Image From File

- **Node Details:**  
  - **Converter**  
    - Type: Code node used to preprocess or transform input data (specific code not detailed).  
    - Configuration: Likely extracts file path or prepares file data for reading.  
    - Input: Output from Local File Trigger (file metadata).  
    - Output: Passes processed file info to Get Image From File.  
    - Edge cases: Expression errors from unexpected input data or file metadata.  

  - **Get Image From File**  
    - Type: Read/Write File node configured to read the image file from disk.  
    - Configuration: Reads the image file from the path provided by the Converter node.  
    - Input: File path from Converter.  
    - Output: The raw image data (binary) passed to AI nodes.  
    - Edge cases: File access errors if file is locked or path invalid; corrupted image files.  

#### 2.3 Image Analysis AI Agent

- **Overview:**  
This block uses GPT-4o to analyze the image deeply. It generates a detailed textual description including theme, style, text phrase, fonts, aesthetics, and complexity.

- **Nodes Involved:**  
  - Analyze Image (OpenAI node)  

- **Node Details:**  
  - **Analyze Image**  
    - Type: OpenAI node calling GPT-4o with image input capability.  
    - Configuration: Uses OpenAI credentials linked to GPT-4o.  
    - Input: Binary image data from Get Image From File.  
    - Output: Structured text including detailed image description and original phrase extracted from the image.  
    - Expressions: System prompt templates likely instruct GPT-4o to analyze specific attributes (art style, phrase, font, etc.).  
    - Edge cases: API quota exhaustion, authentication errors, image too large or unsupported format, model response latency.  
    - Version: Requires OpenAI API with GPT-4o access.  

#### 2.4 Prompt/Text Generation AI Agent

- **Overview:**  
Receives the analysis output and uses GPT-4o to generate five distinct image prompts and nine novel text phrases for further creative use.

- **Nodes Involved:**  
  - Prompt/Text Generator (Langchain Agent)  
  - OpenAI Chat Model (GPT-4o model used by Prompt/Text Generator)  

- **Node Details:**  
  - **OpenAI Chat Model**  
    - Type: Langchain OpenAI Chat Model node.  
    - Configuration: Connected to Prompt/Text Generator node as its language model. Uses same OpenAI credentials as Analyze Image.  
    - Input: Prompts from Prompt/Text Generator node’s internal logic.  
    - Output: Text responses with generated prompts and phrases.  

  - **Prompt/Text Generator**  
    - Type: Langchain Agent node coordinating prompt generation logic.  
    - Configuration: Receives analyzed description and original phrase, instructs OpenAI model to generate:  
      - Five unique image prompts with similar style but distinct creative content.  
      - Nine new text phrases plus the original phrase (total 10).  
    - Input: Text from Analyze Image node.  
    - Output: Combined text with all prompts and phrases for saving.  
    - Edge cases: API call failures, incomplete generation, malformed text output.  

#### 2.5 Output Preparation and Saving

- **Overview:**  
Converts the generated prompt and phrase data into a text file and saves it to a user-designated folder on the local computer.

- **Nodes Involved:**  
  - Convert To Text  
  - Save To File  

- **Node Details:**  
  - **Convert To Text**  
    - Type: Convert To File node.  
    - Configuration: Converts JSON or structured data from Prompt/Text Generator into a plain text file suitable for saving.  
    - Input: Output from Prompt/Text Generator node.  
    - Output: A text file binary data.  
    - Edge cases: Conversion errors if input data malformed.  

  - **Save To File**  
    - Type: Read/Write File node.  
    - Configuration: Save path set to user-specified output folder. Saves the converted text file with a fixed name.  
    - Input: Text file binary from Convert To Text.  
    - Output: None (terminal node).  
    - Notes: User must rename saved file before next run to avoid overwrite.  
    - Edge cases: File write permission issues, path misconfiguration, file locked by another process.

---

### 3. Summary Table

| Node Name           | Node Type                              | Functional Role                        | Input Node(s)          | Output Node(s)         | Sticky Note                                                                                   |
|---------------------|--------------------------------------|-------------------------------------|-----------------------|------------------------|----------------------------------------------------------------------------------------------|
| Local File Trigger   | n8n-nodes-base.localFileTrigger       | Watches local folder for new images | None                  | Converter              |                                                                                              |
| Converter           | n8n-nodes-base.code                   | Prepares file path/data for reading | Local File Trigger    | Get Image From File     |                                                                                              |
| Get Image From File  | n8n-nodes-base.readWriteFile          | Reads image file from disk           | Converter             | Analyze Image           |                                                                                              |
| Analyze Image       | @n8n/n8n-nodes-langchain.openAi       | Analyzes image with GPT-4o           | Get Image From File   | Prompt/Text Generator   |                                                                                              |
| Prompt/Text Generator| @n8n/n8n-nodes-langchain.agent         | Generates new prompts and phrases    | Analyze Image         | Convert To Text         |                                                                                              |
| OpenAI Chat Model   | @n8n/n8n-nodes-langchain.lmChatOpenAi | Language model for prompt generation | Prompt/Text Generator | Prompt/Text Generator   |                                                                                              |
| Convert To Text      | n8n-nodes-base.convertToFile           | Converts AI output to text file      | Prompt/Text Generator | Save To File            |                                                                                              |
| Save To File         | n8n-nodes-base.readWriteFile           | Saves text file to local folder      | Convert To Text       | None                   | After saving, rename file before rerun to prevent overwriting existing file                   |
| Sticky Note1         | n8n-nodes-base.stickyNote              | Informative visual note               | None                  | None                   |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Local File Trigger Node**  
   - Type: `n8n-nodes-base.localFileTrigger`  
   - Set the folder path to monitor (e.g., desktop folder where original images will be saved).  

2. **Add Code Node Named "Converter"**  
   - Type: `n8n-nodes-base.code`  
   - Configure to extract the file path or prepare the trigger data for reading.  
   - Connect input from Local File Trigger node.  

3. **Add Read/Write File Node Named "Get Image From File"**  
   - Type: `n8n-nodes-base.readWriteFile`  
   - Set to read the image file using the path from the Converter node.  
   - Connect input from Converter node.  

4. **Add OpenAI Node Named "Analyze Image"**  
   - Type: `@n8n/n8n-nodes-langchain.openAi`  
   - Configure credentials with OpenAI API key for GPT-4o.  
   - Configure prompt/system message to instruct GPT-4o to analyze image details: theme, style, text phrase, font, aesthetic, complexity.  
   - Connect input from Get Image From File node.  

5. **Add Langchain Agent Node Named "Prompt/Text Generator"**  
   - Type: `@n8n/n8n-nodes-langchain.agent`  
   - Connect an OpenAI Chat Model node as its language model below.  
   - Configure to receive Analyze Image output and instruct GPT-4o to generate:  
     - Five unique image prompts inspired by the description.  
     - Nine new text phrases plus the original phrase.  
   - Connect input from Analyze Image node.  

6. **Add OpenAI Chat Model Node Named "OpenAI Chat Model"**  
   - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
   - Use same API credentials as Analyze Image node.  
   - Connect to Prompt/Text Generator node as language model input.  

7. **Add Convert To File Node Named "Convert To Text"**  
   - Type: `n8n-nodes-base.convertToFile`  
   - Configure to convert JSON or structured prompt data into a plain text file format.  
   - Connect input from Prompt/Text Generator node.  

8. **Add Read/Write File Node Named "Save To File"**  
   - Type: `n8n-nodes-base.readWriteFile`  
   - Set folder path to user’s desired output directory for saving prompt text files.  
   - Set filename (note: fixed name requires manual rename before rerunning).  
   - Connect input from Convert To Text node.  

9. **Set Up Credentials**  
   - Obtain OpenAI API key from platform.openai.com/api-keys.  
   - Configure OpenAI credentials in n8n and link to Analyze Image and OpenAI Chat Model nodes.  

10. **Test Workflow**  
    - Save an image file into the monitored folder to trigger the workflow.  
    - Verify the generated text file appears in the output folder with the expected prompts and phrases.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                           | Context or Link                                                    |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------|
| This workflow uses a community node ([screenshot reference provided]).                                                                                                | Community nodes may require manual installation or update.       |
| Demonstration video available here: [Workflow Demonstration](https://youtu.be/Xco28XW-Ewg)                                                                           | Useful for visual understanding of workflow operation.           |
| GPT-4o costs approximately $0.01-$0.02 per run; ensure your OpenAI account is funded accordingly before heavy use.                                                  | Cost management advice.                                           |
| After the workflow saves the output file, rename it before running again to avoid overwriting the previous results.                                                  | Important operational constraint to prevent data loss.           |
| Ideal users include print-on-demand businesses and digital artists who want to automate prompt engineering for AI image generation.                                | Use case clarification.                                           |

---

This structured document fully captures the workflow's design, logic, node configurations, and usage guidance, enabling advanced users or AI agents to understand, reproduce, and maintain the workflow effectively.
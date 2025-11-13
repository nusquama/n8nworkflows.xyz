Generate Ad Images from Campaign Briefs with GPT-4o and OpenAI Image API

https://n8nworkflows.xyz/workflows/generate-ad-images-from-campaign-briefs-with-gpt-4o-and-openai-image-api-7949


# Generate Ad Images from Campaign Briefs with GPT-4o and OpenAI Image API

### 1. Workflow Overview

This workflow, titled **"Generate Ad Images from Campaign Briefs with GPT-4o and OpenAI Image API"**, automates the creation of advertising images based on campaign briefs using advanced AI models. It leverages an Azure-hosted OpenAI chat model to generate creative prompts and the OpenAI Image API to produce images accordingly. The workflow is ideal for marketing teams, advertisers, or content creators who want to rapidly generate visual assets from text briefs.

The logical flow of the workflow is divided into the following blocks:

- **1.1 Input Reception**: Manual trigger to start the workflow.
- **1.2 AI Prompt Generation**: Uses Azure OpenAI chat model to create detailed image prompts from campaign briefs.
- **1.3 Image Generation**: Calls the OpenAI Image API with generated prompts to create images.
- **1.4 Image Processing & Output**: Splits multiple image outputs, converts them into file format for downstream use.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block provides the entry point into the workflow. It waits for a manual trigger to start processing.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’

- **Node Details:**  
  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Initiates the workflow manually, allowing user control over when the process starts.  
    - Configuration: Default manual trigger with no parameters.  
    - Inputs: None  
    - Outputs: Connects to "Prompt generator for image" node.  
    - Edge Cases: If not triggered, workflow remains idle. No authentication required.  
    - Sub-workflow: None.

#### 2.2 AI Prompt Generation

- **Overview:**  
  This block uses an Azure OpenAI chat model to generate detailed image prompts from the campaign briefs. A Langchain agent orchestrates the prompt creation, ensuring the prompts are well-structured for image generation.

- **Nodes Involved:**  
  - Azure OpenAI Chat Model3  
  - Prompt generator for image  
  - Set Variables

- **Node Details:**  
  - **Azure OpenAI Chat Model3**  
    - Type: Langchain Azure OpenAI Chat Model node  
    - Role: Generates text output (image prompt) from input data using GPT-4o or similar model via Azure OpenAI service.  
    - Configuration: Uses Azure OpenAI credentials and configured chat model parameters (temperature, max tokens, etc.).  
    - Inputs: Receives AI language model input from "Prompt generator for image".  
    - Outputs: Passes generated prompt to "Prompt generator for image" node as AI language model output.  
    - Edge Cases: Possible auth errors if Azure credentials are invalid; API rate limits; model response errors.  
    - Version: Requires Langchain integration compatible with Azure OpenAI.  
    - Sub-workflow: None.

  - **Prompt generator for image**  
    - Type: Langchain Agent node  
    - Role: Constructs and formats the prompt for image generation based on AI chat model output and input data.  
    - Configuration: Uses Langchain agent logic to parse chat model response and prepare prompt variables.  
    - Inputs: Main input from manual trigger; AI language model input from Azure OpenAI Chat Model3.  
    - Outputs: Passes prompt to "Set Variables".  
    - Edge Cases: Expression or parsing errors if chat model output is malformed.  
    - Version: Langchain Agent v2.1.  
    - Sub-workflow: None.

  - **Set Variables**  
    - Type: Set node  
    - Role: Sets or modifies workflow variables, preparing the final prompt text and parameters for the image generation API.  
    - Configuration: Likely sets JSON or string variables such as prompt text, image count, size, etc.  
    - Inputs: Receives prompt data from "Prompt generator for image".  
    - Outputs: Connects to "OpenAI - Generate Image".  
    - Edge Cases: Errors setting variables if expressions reference missing data.  
    - Version: n8n base node v3.4.  
    - Sub-workflow: None.

#### 2.3 Image Generation

- **Overview:**  
  This block calls the OpenAI Image generation API using the prompt generated earlier. It handles the HTTP request to generate ad images based on the prompt.

- **Nodes Involved:**  
  - OpenAI - Generate Image

- **Node Details:**  
  - **OpenAI - Generate Image**  
    - Type: HTTP Request node  
    - Role: Sends a request to the OpenAI Image API (DALL·E or similar) to generate images from the prompt.  
    - Configuration:  
      - HTTP Method: POST  
      - URL: OpenAI Image API endpoint  
      - Authentication: Uses OpenAI credentials (API key)  
      - Request Body: Includes prompt, number of images, size, and response format parameters.  
    - Inputs: Receives prepared prompt and parameters from "Set Variables".  
    - Outputs: Returns image URLs or base64 data to "Separate Image Outputs".  
    - Edge Cases: Possible API errors such as auth failure, rate limiting, invalid prompt errors, or network timeouts.  
    - Version: HTTP Request v4.2.  
    - Sub-workflow: None.

#### 2.4 Image Processing & Output

- **Overview:**  
  This block processes the multiple image URLs returned by the OpenAI API. It separates multiple images into individual items and converts them into file objects usable later in the workflow or for export.

- **Nodes Involved:**  
  - Separate Image Outputs  
  - Convert to File

- **Node Details:**  
  - **Separate Image Outputs**  
    - Type: Split Out node  
    - Role: Splits the array of image URLs into separate workflow items for parallel or sequential processing.  
    - Configuration: Default split on the array output from the HTTP Request node.  
    - Inputs: Receives array of images from "OpenAI - Generate Image".  
    - Outputs: Passes individual image data to "Convert to File".  
    - Edge Cases: If response data is not an array, splitting fails or results in empty output.  
    - Version: v1.  
    - Sub-workflow: None.

  - **Convert to File**  
    - Type: Convert To File node  
    - Role: Converts image URLs or base64 data into n8n file format for downstream processing or export.  
    - Configuration: Default conversion settings, likely using URL or base64 input.  
    - Inputs: Receives single image data from "Separate Image Outputs".  
    - Outputs: Final file object for further use or storage.  
    - Edge Cases: If image URL is invalid or unreachable, file conversion fails; network errors possible.  
    - Version: v1.1.  
    - Sub-workflow: None.

---

### 3. Summary Table

| Node Name                 | Node Type                                 | Functional Role                      | Input Node(s)                  | Output Node(s)              | Sticky Note |
|---------------------------|-------------------------------------------|------------------------------------|-------------------------------|-----------------------------|-------------|
| When clicking ‘Execute workflow’ | Manual Trigger                            | Workflow entry trigger              | -                             | Prompt generator for image   |             |
| Azure OpenAI Chat Model3   | Langchain Azure OpenAI Chat Model          | Generates AI text prompts           | Prompt generator for image (ai_languageModel) | Prompt generator for image (ai_languageModel) |             |
| Prompt generator for image | Langchain Agent                            | Constructs image generation prompts | When clicking ‘Execute workflow’, Azure OpenAI Chat Model3 | Set Variables              |             |
| Set Variables             | Set Node                                  | Prepares prompt and parameters      | Prompt generator for image     | OpenAI - Generate Image      |             |
| OpenAI - Generate Image    | HTTP Request                             | Calls OpenAI Image API to generate images | Set Variables                 | Separate Image Outputs       |             |
| Separate Image Outputs     | Split Out                                | Splits image array into single items | OpenAI - Generate Image        | Convert to File              |             |
| Convert to File            | Convert To File                          | Converts image URLs/data to file objects | Separate Image Outputs         | -                           |             |
| Sticky Note               | Sticky Note                              | Comments or annotations             | -                             | -                           |             |
| Sticky Note1              | Sticky Note                              | Comments or annotations             | -                             | -                           |             |
| Sticky Note2              | Sticky Note                              | Comments or annotations             | -                             | -                           |             |
| Sticky Note3              | Sticky Note                              | Comments or annotations             | -                             | -                           |             |
| Sticky Note4              | Sticky Note                              | Comments or annotations             | -                             | -                           |             |
| Sticky Note5              | Sticky Note                              | Comments or annotations             | -                             | -                           |             |
| Sticky Note6              | Sticky Note                              | Comments or annotations             | -                             | -                           |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a **Manual Trigger** node named "When clicking ‘Execute workflow’".  
   - Default settings; no parameters needed.

2. **Add Azure OpenAI Chat Model Node**  
   - Add a **Langchain Azure OpenAI Chat Model** node named "Azure OpenAI Chat Model3".  
   - Configure with your Azure OpenAI credentials: Endpoint, API key, deployment/model name (e.g., GPT-4o).  
   - Set model parameters: temperature, max tokens, etc., as desired for prompt creativity.  
   - Connect the **ai_languageModel** input of this node from the "Prompt generator for image" node (see next step).

3. **Add Langchain Agent Node for Prompt Generation**  
   - Add a **Langchain Agent** node named "Prompt generator for image".  
   - Configure the agent to receive the manual trigger input and the AI language model output from "Azure OpenAI Chat Model3".  
   - Define agent logic to generate creative prompt text from campaign briefs.  
   - Connect the **main** output of the manual trigger node to this node’s main input.  
   - Connect the **ai_languageModel** output of "Azure OpenAI Chat Model3" back to this node’s **ai_languageModel** input.

4. **Add Set Node to Prepare Variables**  
   - Add a **Set** node named "Set Variables".  
   - Configure it to store the final prompt text and any additional parameters needed by the OpenAI Image API such as number of images and image size.  
   - Connect the main output of "Prompt generator for image" to this node.

5. **Add HTTP Request Node for OpenAI Image API**  
   - Add an **HTTP Request** node named "OpenAI - Generate Image".  
   - Configure it with:  
     - HTTP Method: POST  
     - URL: OpenAI Image generation endpoint (e.g., https://api.openai.com/v1/images/generations)  
     - Authentication: Use OpenAI API credentials (API Key via n8n credentials)  
     - Body Parameters: JSON with the prompt from "Set Variables", number of images, size (e.g., 1024x1024), response format.  
   - Connect the output of "Set Variables" to this node.

6. **Add Split Out Node to Separate Images**  
   - Add a **Split Out** node named "Separate Image Outputs".  
   - Configure to split the array of images from the HTTP response.  
   - Connect the output of "OpenAI - Generate Image" to this node.

7. **Add Convert to File Node**  
   - Add a **Convert To File** node named "Convert to File".  
   - Configure to convert image URLs or base64 image data to n8n file objects.  
   - Connect the output of "Separate Image Outputs" to this node.

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                                                 |
|------------------------------------------------------------------------------|----------------------------------------------------------------|
| The workflow uses Langchain nodes integrated in n8n for advanced AI prompt management. | Langchain integration documentation in n8n                     |
| Azure OpenAI chat model requires an Azure subscription with OpenAI service enabled. | https://azure.microsoft.com/en-us/services/openai-service/      |
| OpenAI Image API endpoint and parameters must be kept up to date to avoid breaking changes. | https://platform.openai.com/docs/api-reference/images          |
| Ensure API keys and credentials are securely stored in n8n credential store. | n8n credential management best practices                        |
| This workflow assumes the campaign brief input is provided when triggering manually or via an upstream node. | Input format depends on use case                                |

---

**Disclaimer:**  
The text provided is exclusively extracted from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.
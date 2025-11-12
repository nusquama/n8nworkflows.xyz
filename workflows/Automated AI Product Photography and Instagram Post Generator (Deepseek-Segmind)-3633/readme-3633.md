Automated AI Product Photography and Instagram Post Generator (Deepseek/Segmind)

https://n8nworkflows.xyz/workflows/automated-ai-product-photography-and-instagram-post-generator--deepseek-segmind--3633


# Automated AI Product Photography and Instagram Post Generator (Deepseek/Segmind)

### 1. Workflow Overview

This workflow automates the generation of professional-grade product photography and Instagram posts using AI, delivering the results to a Telegram chat for approval. It is designed primarily for e-commerce owners, dropshippers, social media managers, and small businesses who want to automate and streamline content creation for marketing purposes.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger & Input Setup:** Initiates the workflow on a defined schedule and sets up essential inputs such as the product image URL, Segmind API key, Telegram Chat ID, and user preferences.
- **1.2 Product Image Analysis:** Uses OpenAI to analyze the input product image URL and extract descriptive details.
- **1.3 Human Model Decision Branch:** Determines if the generated product photo should include a human model or not, based on user preference.
- **1.4 AI Prompt & Instagram Caption Generation:** Uses Langchain agents with OpenAI Chat models to generate prompts for product photography and Instagram captions.
- **1.5 Random Seed Generation & Image Request:** Generates a random seed number to ensure image uniqueness and sends a request to Segmind API to generate the product photo.
- **1.6 Image Retrieval & Download:** Waits for the image generation to complete, retrieves the generated image URL, and downloads the image.
- **1.7 Telegram Delivery:** Sends the generated product photo and Instagram post caption to the configured Telegram chat for validation and approval.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger & Input Setup

- **Overview:** This block triggers the workflow on a user-defined schedule and sets up all necessary inputs including API keys, product image URL, Telegram chat ID, and user preferences.
- **Nodes Involved:** Schedule Trigger, SegmindAPIKey, ImageInstruction, InputYourImageURL, Telegram ChatID
- **Node Details:**

  - **Schedule Trigger**
    - Type: Schedule Trigger
    - Role: Initiates the workflow at defined intervals (e.g., hourly, daily)
    - Configuration: User sets interval and time unit in node parameters
    - Inputs: None (trigger node)
    - Outputs: SegmindAPIKey node
    - Edge Cases: Misconfigured schedule may cause no triggers or too frequent triggers

  - **SegmindAPIKey**
    - Type: Set
    - Role: Stores the Segmind API key for authentication with the image generation API
    - Configuration: User pastes API key into a field named `Value`
    - Inputs: Schedule Trigger
    - Outputs: ImageInstruction node
    - Edge Cases: Missing or invalid API key will cause authentication failures downstream

  - **ImageInstruction**
    - Type: Set
    - Role: Stores user preference whether to include a human model in the generated image
    - Configuration: Boolean field `HumanModel` set to true or false by user
    - Inputs: SegmindAPIKey
    - Outputs: InputYourImageURL node
    - Edge Cases: Invalid or missing value may cause branching errors later

  - **InputYourImageURL**
    - Type: Set
    - Role: Stores the product image URL input by the user
    - Configuration: URL string field `Value`
    - Inputs: ImageInstruction
    - Outputs: Telegram ChatID node
    - Edge Cases: Invalid URL format or inaccessible URL will cause failures in image analysis

  - **Telegram ChatID**
    - Type: Set
    - Role: Stores the Telegram chat ID where messages will be sent
    - Configuration: String field `Value` containing chat ID
    - Inputs: InputYourImageURL
    - Outputs: Get image description node
    - Edge Cases: Incorrect chat ID will cause message delivery failures

---

#### 2.2 Product Image Analysis

- **Overview:** Uses OpenAI to analyze the product image URL and generate a descriptive text about the product.
- **Nodes Involved:** Get image description
- **Node Details:**

  - **Get image description**
    - Type: OpenAI (Langchain)
    - Role: Generates a textual description of the product based on the image URL
    - Configuration: Uses OpenAI model configured in credentials; prompt likely includes the image URL for analysis
    - Inputs: Telegram ChatID
    - Outputs: Check if an Human Model is required node
    - Edge Cases: API rate limits, invalid image URL, or prompt failures can cause errors

---

#### 2.3 Human Model Decision Branch

- **Overview:** Branches the workflow based on whether the user wants a human model included in the product photo.
- **Nodes Involved:** Check if an Human Model is required, WithHumanModel, NoHumanModel
- **Node Details:**

  - **Check if an Human Model is required**
    - Type: If
    - Role: Evaluates the boolean `HumanModel` value to decide the next path
    - Configuration: Condition checks if `HumanModel` is true or false
    - Inputs: Get image description
    - Outputs: WithHumanModel (if true), NoHumanModel (if false)
    - Edge Cases: Missing or malformed boolean value can cause incorrect branching

  - **WithHumanModel**
    - Type: Set
    - Role: Prepares data for AI prompt generation including human model instructions
    - Configuration: Sets variables or flags indicating human model inclusion
    - Inputs: Check if an Human Model is required (true branch)
    - Outputs: AI Agent1 node
    - Edge Cases: Misconfiguration may cause prompt generation errors

  - **NoHumanModel**
    - Type: Set
    - Role: Prepares data for AI prompt generation excluding human model instructions
    - Configuration: Sets variables or flags indicating no human model
    - Inputs: Check if an Human Model is required (false branch)
    - Outputs: AI Agent1 node
    - Edge Cases: Misconfiguration may cause prompt generation errors

---

#### 2.4 AI Prompt & Instagram Caption Generation

- **Overview:** Generates detailed AI prompts for product photography and Instagram post captions using Langchain agents and OpenAI chat models.
- **Nodes Involved:** AI Agent1, OpenAI Chat Model1, Structured Output Parser1, GenerateRandomSeedNumber
- **Node Details:**

  - **AI Agent1**
    - Type: Langchain Agent
    - Role: Coordinates prompt generation for both product photography and Instagram captions
    - Configuration: Uses OpenAI Chat Model1 as language model; input includes product description and human model preference
    - Inputs: WithHumanModel or NoHumanModel
    - Outputs: GenerateRandomSeedNumber
    - Edge Cases: API errors, prompt failures, or unexpected input format

  - **OpenAI Chat Model1**
    - Type: Langchain OpenAI Chat Model
    - Role: Provides the underlying language model for AI Agent1
    - Configuration: Uses OpenAI credentials; model parameters can be customized
    - Inputs: AI Agent1 (as language model)
    - Outputs: AI Agent1 (as AI language model output)
    - Edge Cases: API rate limits, invalid credentials

  - **Structured Output Parser1**
    - Type: Langchain Structured Output Parser
    - Role: Parses AI Agent1 output into structured data for downstream use
    - Configuration: Uses predefined schema to extract prompt and caption
    - Inputs: AI Agent1
    - Outputs: AI Agent1 (as output parser)
    - Edge Cases: Parsing errors if AI output format changes

  - **GenerateRandomSeedNumber**
    - Type: Code
    - Role: Generates a random seed number to ensure uniqueness in image generation
    - Configuration: Custom JavaScript code generating a random integer or string
    - Inputs: AI Agent1
    - Outputs: Set seed field
    - Edge Cases: Code errors or unexpected input

---

#### 2.5 Random Seed Generation & Image Request

- **Overview:** Sets the seed field for image generation and sends the image generation request to Segmind API.
- **Nodes Involved:** Set seed field, GetImage_Segmind
- **Node Details:**

  - **Set seed field**
    - Type: Set
    - Role: Adds the generated random seed number to the request payload
    - Configuration: Sets a field with the seed value from GenerateRandomSeedNumber
    - Inputs: GenerateRandomSeedNumber
    - Outputs: GetImage_Segmind
    - Edge Cases: Missing seed value or incorrect field name

  - **GetImage_Segmind**
    - Type: HTTP Request
    - Role: Sends the prompt, product image URL, and seed to Segmind API to generate the product photo
    - Configuration: HTTP POST request with authentication using Segmind API key; payload includes prompt and seed
    - Inputs: Set seed field
    - Outputs: Wait 5 minutes
    - Edge Cases: API authentication failure, network timeout, invalid payload

---

#### 2.6 Image Retrieval & Download

- **Overview:** Waits for image generation to complete, retrieves the generated image URL, and downloads the image.
- **Nodes Involved:** Wait 5 minutes, RetrieveGeneratedImage, RetrieveImageURL, DownloadImage
- **Node Details:**

  - **Wait 5 minutes**
    - Type: Wait
    - Role: Pauses workflow to allow Segmind API to complete image generation
    - Configuration: Fixed 5-minute wait period
    - Inputs: GetImage_Segmind
    - Outputs: RetrieveGeneratedImage
    - Edge Cases: Insufficient wait time may cause retrieval of incomplete data

  - **RetrieveGeneratedImage**
    - Type: HTTP Request
    - Role: Queries Segmind API to get the URL of the generated image
    - Configuration: HTTP GET request with authentication
    - Inputs: Wait 5 minutes
    - Outputs: RetrieveImageURL
    - Edge Cases: API errors, image not ready, network issues

  - **RetrieveImageURL**
    - Type: Set
    - Role: Extracts and sets the image URL from the API response for download
    - Configuration: Sets a field with the image URL
    - Inputs: RetrieveGeneratedImage
    - Outputs: DownloadImage
    - Edge Cases: Missing or malformed URL in response

  - **DownloadImage**
    - Type: HTTP Request
    - Role: Downloads the generated image file for sending to Telegram
    - Configuration: HTTP GET request to image URL; response type set to binary
    - Inputs: RetrieveImageURL
    - Outputs: SendProductPhotography
    - Edge Cases: Download failure, invalid URL, network issues

---

#### 2.7 Telegram Delivery

- **Overview:** Sends the generated product photo and Instagram caption to Telegram chat for review.
- **Nodes Involved:** SendProductPhotography, SendInstagramPost, SendImageURL
- **Node Details:**

  - **SendProductPhotography**
    - Type: Telegram
    - Role: Sends the downloaded product photo as a message to Telegram chat
    - Configuration: Uses Telegram Bot credentials; configured with target Chat ID; sends photo as binary data
    - Inputs: DownloadImage
    - Outputs: SendInstagramPost
    - Edge Cases: Invalid chat ID, Telegram API errors, network issues

  - **SendInstagramPost**
    - Type: Telegram
    - Role: Sends the generated Instagram post caption text to Telegram chat
    - Configuration: Uses Telegram Bot credentials; configured with target Chat ID; sends text message
    - Inputs: SendProductPhotography
    - Outputs: SendImageURL
    - Edge Cases: Same as above

  - **SendImageURL**
    - Type: Telegram
    - Role: Optionally sends the direct URL of the generated image to Telegram chat
    - Configuration: Uses Telegram Bot credentials; configured with target Chat ID; sends text message with URL
    - Inputs: SendInstagramPost
    - Outputs: None (end of workflow)
    - Edge Cases: Same as above

---

### 3. Summary Table

| Node Name               | Node Type                        | Functional Role                              | Input Node(s)               | Output Node(s)              | Sticky Note                          |
|-------------------------|---------------------------------|----------------------------------------------|-----------------------------|-----------------------------|------------------------------------|
| Schedule Trigger        | Schedule Trigger                | Starts workflow on defined schedule          | None                        | SegmindAPIKey               |                                    |
| SegmindAPIKey           | Set                            | Stores Segmind API key                        | Schedule Trigger            | ImageInstruction            |                                    |
| ImageInstruction        | Set                            | Stores human model preference                 | SegmindAPIKey               | InputYourImageURL           |                                    |
| InputYourImageURL       | Set                            | Stores product image URL                      | ImageInstruction            | Telegram ChatID             |                                    |
| Telegram ChatID         | Set                            | Stores Telegram chat ID                       | InputYourImageURL           | Get image description       |                                    |
| Get image description   | OpenAI (Langchain)              | Generates product description from image URL | Telegram ChatID             | Check if an Human Model is required |                                    |
| Check if an Human Model is required | If                     | Branches based on human model preference     | Get image description       | WithHumanModel, NoHumanModel |                                    |
| WithHumanModel          | Set                            | Prepares data for prompt with human model    | Check if an Human Model is required (true) | AI Agent1                  |                                    |
| NoHumanModel            | Set                            | Prepares data for prompt without human model | Check if an Human Model is required (false) | AI Agent1                  |                                    |
| AI Agent1               | Langchain Agent                | Generates AI prompts and Instagram captions  | WithHumanModel, NoHumanModel | GenerateRandomSeedNumber    |                                    |
| OpenAI Chat Model1      | Langchain OpenAI Chat Model    | Provides language model for AI Agent1        | AI Agent1 (as language model) | AI Agent1 (as output)       |                                    |
| Structured Output Parser1 | Langchain Output Parser       | Parses AI Agent1 output into structured data | AI Agent1                   | AI Agent1 (output parser)   |                                    |
| GenerateRandomSeedNumber | Code                          | Generates random seed for image uniqueness   | AI Agent1                   | Set seed field              |                                    |
| Set seed field          | Set                            | Adds seed to image generation request        | GenerateRandomSeedNumber    | GetImage_Segmind            |                                    |
| GetImage_Segmind        | HTTP Request                   | Sends image generation request to Segmind API | Set seed field              | Wait 5 minutes              |                                    |
| Wait 5 minutes          | Wait                           | Pauses workflow to allow image generation    | GetImage_Segmind            | RetrieveGeneratedImage      |                                    |
| RetrieveGeneratedImage  | HTTP Request                   | Retrieves generated image URL from Segmind   | Wait 5 minutes              | RetrieveImageURL            |                                    |
| RetrieveImageURL        | Set                            | Extracts image URL from API response          | RetrieveGeneratedImage      | DownloadImage               |                                    |
| DownloadImage           | HTTP Request                   | Downloads generated image file                | RetrieveImageURL            | SendProductPhotography      |                                    |
| SendProductPhotography  | Telegram                       | Sends product photo to Telegram chat          | DownloadImage               | SendInstagramPost           |                                    |
| SendInstagramPost       | Telegram                       | Sends Instagram caption to Telegram chat      | SendProductPhotography      | SendImageURL                |                                    |
| SendImageURL            | Telegram                       | Sends image URL to Telegram chat               | SendInstagramPost           | None                       |                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**
   - Set the interval and time unit (e.g., every 1 hour) to define how often the workflow runs.
   - Connect its output to the next node.

2. **Create a Set node named `SegmindAPIKey`**
   - Add a field `Value` and paste your Segmind API key.
   - Connect Schedule Trigger output to this node.

3. **Create a Set node named `ImageInstruction`**
   - Add a boolean field `HumanModel`.
   - Set to `true` if you want human models in photos; otherwise `false`.
   - Connect `SegmindAPIKey` output to this node.

4. **Create a Set node named `InputYourImageURL`**
   - Add a field `Value` and enter your product image URL.
   - Connect `ImageInstruction` output to this node.

5. **Create a Set node named `Telegram ChatID`**
   - Add a field `Value` with your Telegram chat ID.
   - Connect `InputYourImageURL` output to this node.

6. **Create an OpenAI node named `Get image description`**
   - Configure with your OpenAI credentials.
   - Set prompt to analyze the product image URL and generate a product description.
   - Connect `Telegram ChatID` output to this node.

7. **Create an If node named `Check if an Human Model is required`**
   - Condition: Check if `HumanModel` field is `true`.
   - Connect `Get image description` output to this node.

8. **Create two Set nodes: `WithHumanModel` and `NoHumanModel`**
   - `WithHumanModel`: Set variables or flags indicating human model inclusion.
   - `NoHumanModel`: Set variables or flags indicating no human model.
   - Connect `Check if an Human Model is required` true output to `WithHumanModel`.
   - Connect false output to `NoHumanModel`.

9. **Create a Langchain Agent node named `AI Agent1`**
   - Configure to use OpenAI Chat Model1 as language model.
   - Input includes product description and human model preference.
   - Connect outputs of both `WithHumanModel` and `NoHumanModel` to this node.

10. **Create a Langchain OpenAI Chat Model node named `OpenAI Chat Model1`**
    - Configure with OpenAI credentials.
    - Connect as language model input to `AI Agent1`.

11. **Create a Langchain Structured Output Parser node named `Structured Output Parser1`**
    - Configure to parse AI Agent1 output into structured prompt and caption.
    - Connect output parser input to `AI Agent1`.

12. **Create a Code node named `GenerateRandomSeedNumber`**
    - Add JavaScript code to generate a random seed number (e.g., random integer).
    - Connect `AI Agent1` output to this node.

13. **Create a Set node named `Set seed field`**
    - Add a field to hold the seed number from `GenerateRandomSeedNumber`.
    - Connect `GenerateRandomSeedNumber` output to this node.

14. **Create an HTTP Request node named `GetImage_Segmind`**
    - Configure POST request to Segmind API endpoint.
    - Include prompt, product image URL, and seed in payload.
    - Use Segmind API key for authentication.
    - Connect `Set seed field` output to this node.

15. **Create a Wait node named `Wait 5 minutes`**
    - Set wait time to 5 minutes.
    - Connect `GetImage_Segmind` output to this node.

16. **Create an HTTP Request node named `RetrieveGeneratedImage`**
    - Configure GET request to Segmind API to retrieve generated image URL.
    - Use API key for authentication.
    - Connect `Wait 5 minutes` output to this node.

17. **Create a Set node named `RetrieveImageURL`**
    - Extract and set the image URL from the API response.
    - Connect `RetrieveGeneratedImage` output to this node.

18. **Create an HTTP Request node named `DownloadImage`**
    - Configure GET request to the image URL.
    - Set response type to binary to download image file.
    - Connect `RetrieveImageURL` output to this node.

19. **Create a Telegram node named `SendProductPhotography`**
    - Configure with Telegram Bot credentials.
    - Set to send photo to the chat ID.
    - Connect `DownloadImage` output to this node.

20. **Create a Telegram node named `SendInstagramPost`**
    - Configure with Telegram Bot credentials.
    - Set to send Instagram caption text to the chat ID.
    - Connect `SendProductPhotography` output to this node.

21. **Create a Telegram node named `SendImageURL`**
    - Configure with Telegram Bot credentials.
    - Set to send the image URL text to the chat ID.
    - Connect `SendInstagramPost` output to this node.

22. **Activate the workflow**
    - Ensure all credentials are configured.
    - Test the workflow with valid inputs.
    - Activate for scheduled runs.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                      |
|---------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Segmind API key required: https://cloud.segmind.com/console/api-keys                                          | Segmind API key setup for image generation                                                         |
| Telegram Bot setup video guidance available                                                                   | Helps configure Telegram Bot credentials and chat ID                                               |
| Estimated cost per product photography and Instagram post generation is approximately $0.10                    | Useful for budgeting and cost estimation                                                            |
| Workflow categories: Marketing, Social Media, E-commerce, Automation, AI, Content Creation, Product Photography | Tags for organizing and searching workflows                                                        |
| Customization tips: Modify AI Agent1 prompt for photography style and Instagram caption tone                   | Allows tailoring output to specific brand aesthetics and marketing goals                            |
| Wait node set to 5 minutes to allow Segmind image generation completion                                        | Adjust wait time if API response times vary                                                        |
| Use of Langchain agents and OpenAI chat models for advanced AI prompt generation                               | Enables sophisticated and context-aware content creation                                           |

---

This structured documentation provides a comprehensive understanding of the workflow, enabling reproduction, modification, and troubleshooting by advanced users and AI agents alike.
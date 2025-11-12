Create AI Videos with Scripts, Images & HeyGen Avatars (🔥 LIMITED-TIME OFFER)

https://n8nworkflows.xyz/workflows/create-ai-videos-with-scripts--images---heygen-avatars-----limited-time-offer--4569


# Create AI Videos with Scripts, Images & HeyGen Avatars (🔥 LIMITED-TIME OFFER)

---

## 1. Workflow Overview

This n8n workflow is designed to automate the creation of AI-generated videos using scripts, images, and avatars from HeyGen, a platform for AI avatars. It targets content creators, marketers, and developers who want to generate videos programmatically by leveraging AI for script generation, image creation, avatar video synthesis, and captioning.

The workflow is structured into the following logical blocks:

- **1.1 Input Reception & Validation:** Receiving requests via webhook and deciding whether to process them.
- **1.2 Script Generation & Processing:** Using AI language models to create or improve video scripts.
- **1.3 Scene Handling & Image Generation:** Splitting scripts into scenes, generating images for each scene, and improving prompts for image generation.
- **1.4 Video Creation via Multiple Services:** Creating videos using Runway, HeyGen avatars, and json2video platforms depending on script type and background conditions.
- **1.5 Caption Generation:** Generating captions for videos using AI and handling caption-related errors.
- **1.6 Error Handling & Workflow Control:** Managing errors from various nodes and workflows, including Baserow database logging.
- **1.7 Sub-Workflow Executions:** Delegating complex or reusable tasks to sub-workflows to keep the main workflow manageable.

---

## 2. Block-by-Block Analysis

### 1.1 Input Reception & Validation

**Overview:**  
Receives the initial trigger via webhook, sets the request body, and decides if processing should proceed.

**Nodes Involved:**  
- Webhook  
- Body (Set node)  
- Should Process? (If node)  
- Switch ScriptType (Switch node)  
- Baserow Processing (Baserow node)

**Node Details:**

- **Webhook**  
  - Type: Webhook (Trigger)  
  - Role: Entry point; receives incoming request data to start the workflow.  
  - Configuration: Standard webhook listening on a specified URL.  
  - Inputs: External HTTP requests.  
  - Outputs: Passes data to Body node.

- **Body**  
  - Type: Set  
  - Role: Sets or formats the incoming data for later nodes.  
  - Configuration: Likely extracts or formats relevant request body parameters.  
  - Inputs: From Webhook node.  
  - Outputs: To Should Process? node.

- **Should Process?**  
  - Type: If  
  - Role: Decides whether to proceed based on input conditions (e.g., validation, flags).  
  - Configuration: Conditional checks on input data.  
  - Inputs: From Body node.  
  - Outputs:  
    - True: To Switch ScriptType node (continue processing).  
    - False: To Baserow Processing node (possibly logging or alternate processing).

- **Switch ScriptType**  
  - Type: Switch  
  - Role: Routes workflow based on script type field (e.g., manual vs automatic script).  
  - Configuration: Checks a field that indicates script source or style.  
  - Inputs: From Should Process? node (true branch).  
  - Outputs:  
    - To If node (for manual script path).  
    - To Basic LLM Chain node (for automatic script path).

- **Baserow Processing**  
  - Type: Baserow (Database)  
  - Role: Logs or processes data when Should Process? returns false.  
  - Configuration: Interacts with Baserow database for record creation or update.  
  - Inputs: From Should Process? node (false branch).  
  - Outputs: None specified (likely end or error handling).

**Version Requirements:** None specific; standard n8n nodes and Baserow integration.

**Potential Failures:**  
- Webhook misconfiguration or network issues.  
- Conditional logic errors in If or Switch nodes.  
- Baserow auth or connectivity issues.

---

### 1.2 Script Generation & Processing

**Overview:**  
Generates or improves scripts using OpenAI language models, parses structured output, updates scripts in database, and manages manual or automatic script flows.

**Nodes Involved:**  
- If  
- Basic LLM Chain  
- Basic LLM Chain Manual  
- OpenAI Chat Model  
- OpenAI Chat Model1  
- Structured Output Parser  
- Update Script (Baserow)  
- Execute Workflow4  
- Execute Workflow

**Node Details:**

- **If**  
  - Type: If  
  - Role: Branches workflow depending on script type or other criteria.  
  - Inputs: From Switch ScriptType node.  
  - Outputs:  
    - True: To Basic LLM Chain Manual node.  
    - False: To Execute Workflow node.

- **Basic LLM Chain**  
  - Type: LangChain LLM Chain (OpenAI)  
  - Role: Automatically generate or process scripts using AI.  
  - Configuration: Uses OpenAI Chat Model as language model and Structured Output Parser for parsing AI responses.  
  - Inputs: From Switch ScriptType node or If node.  
  - Outputs: To Scenes Mapping and Update Script nodes, and Execute Workflow4 node.

- **Basic LLM Chain Manual**  
  - Type: LangChain LLM Chain (OpenAI)  
  - Role: Processes manually provided scripts with AI improvements.  
  - Inputs: From If node.  
  - Outputs: To Update Script and Scenes Mapping nodes, and Execute Workflow4 node.

- **OpenAI Chat Model** & **OpenAI Chat Model1**  
  - Type: LangChain AI Language Model (OpenAI)  
  - Role: Provide AI model for script generation and improvements.  
  - Inputs: Connected internally to Basic LLM Chain nodes as AI language model.  
  - Outputs: AI generated content to Basic LLM Chain nodes.

- **Structured Output Parser**  
  - Type: LangChain Output Parser  
  - Role: Parses structured output from AI responses into usable data formats.  
  - Inputs: From Basic LLM Chain and Basic LLM Chain Manual nodes.  
  - Outputs: Parsed data for further processing.

- **Update Script**  
  - Type: Baserow  
  - Role: Updates script records in the Baserow database with AI-generated or improved content.  
  - Inputs: From Basic LLM Chain and Basic LLM Chain Manual.  
  - Outputs: None specified.

- **Execute Workflow4**  
  - Type: Execute Workflow  
  - Role: Triggers a sub-workflow for further script or scene processing.  
  - Inputs: From Basic LLM Chain and Basic LLM Chain Manual.  
  - Outputs: None specified.

- **Execute Workflow**  
  - Type: Execute Workflow  
  - Role: Triggers an alternative sub-workflow (possibly for manual scripts).  
  - Inputs: From If node.  
  - Outputs: None specified.

**Edge Cases / Failures:**  
- AI rate limits or auth errors with OpenAI credentials.  
- Parsing failures if AI output is malformed.  
- Baserow update failures (auth, invalid data).  
- Sub-workflow failures.

---

### 1.3 Scene Handling & Image Generation

**Overview:**  
Maps scenes from scripts, batches scene processing, generates images using external APIs (Leo), improves prompts, and waits for asynchronous image generation completion.

**Nodes Involved:**  
- Scenes Mapping (Set)  
- loop_over_scenes (SplitInBatches)  
- Split Out (SplitOut)  
- Leo - Improve Prompt (HTTP Request)  
- Leo - Generate Image (HTTP Request)  
- Leo - Get imageId (HTTP Request)  
- Wait1 (Wait)  
- BackgroundType (If)  
- output image (Set)  
- Execute Workflow3  
- Execute Workflow5

**Node Details:**

- **Scenes Mapping**  
  - Type: Set  
  - Role: Maps or formats scenes extracted from the script for processing.  
  - Inputs: From Basic LLM Chain.  
  - Outputs: To Split Out node.

- **Split Out**  
  - Type: SplitOut  
  - Role: Splits array of scenes into individual items for batch processing.  
  - Inputs: From Scenes Mapping.  
  - Outputs: To loop_over_scenes.

- **loop_over_scenes**  
  - Type: SplitInBatches  
  - Role: Processes each scene in batches to control throughput and API usage.  
  - Inputs: From output node or Split Out.  
  - Outputs: To If_with_avatar and Leo - Improve Prompt nodes.

- **Leo - Improve Prompt**  
  - Type: HTTP Request  
  - Role: Sends prompts to Leo API to refine image generation prompts.  
  - Inputs: From loop_over_scenes.  
  - Outputs: To Leo - Generate Image and Execute Workflow5 nodes.

- **Leo - Generate Image**  
  - Type: HTTP Request  
  - Role: Requests image generation from Leo API based on improved prompts.  
  - Inputs: From Leo - Improve Prompt.  
  - Outputs: To Wait1 node and Execute Workflow3 sub-workflow.

- **Wait1**  
  - Type: Wait  
  - Role: Pauses workflow to allow asynchronous image generation to complete.  
  - Inputs: From Leo - Generate Image.  
  - Outputs: To Leo - Get imageId node.

- **Leo - Get imageId**  
  - Type: HTTP Request  
  - Role: Queries Leo API to retrieve the generated image ID after processing.  
  - Inputs: From Wait1 node.  
  - Outputs: To BackgroundType node.

- **BackgroundType**  
  - Type: If  
  - Role: Checks background type or other conditions to decide downstream video creation path.  
  - Inputs: From Leo - Get imageId.  
  - Outputs:  
    - True: To Runway - Create Video node.  
    - False: To output image node.

- **output image**  
  - Type: Set  
  - Role: Sets or formats image data for further processing.  
  - Inputs: From BackgroundType node.  
  - Outputs: To loop_over_scenes for further batch processing.

- **Execute Workflow3** and **Execute Workflow5**  
  - Type: Execute Workflow  
  - Role: Trigger sub-workflows for additional image or prompt processing.  
  - Inputs: From Leo - Generate Image and Leo - Improve Prompt respectively.  
  - Outputs: None specified.

**Edge Cases / Failures:**  
- HTTP request failures or timeouts to Leo API.  
- Wait node timeout or webhook issues.  
- Conditional logic errors in BackgroundType.  
- Sub-workflow failure.

---

### 1.4 Video Creation via Multiple Services

**Overview:**  
Creates videos using Runway, HeyGen avatars, or json2video platform depending on the script and scene data, includes polling for video readiness, and error handling for video creation.

**Nodes Involved:**  
- Runway - Create Video (HTTP Request)  
- Runway - Get Video (HTTP Request)  
- Wait2 (Wait)  
- Execute Workflow2  
- HeyGen (HTTP Request)  
- HeyGen : Check Video (HTTP Request)  
- Wait4 (Wait)  
- heygen_response (Switch)  
- Code Heygen (Code)  
- json2video : Video Rendering (HTTP Request)  
- json2video : Check Video Rendering (HTTP Request)  
- Wait (Wait)  
- j2v_response (Switch)  
- Execute Workflow6  
- Execute Workflow2  
- json2video Execute ERROR  
- heygen Execute ERROR  
- heygen Execute ERROR2  
- json2video Execute ERROR1

**Node Details:**

- **Runway - Create Video**  
  - Type: HTTP Request  
  - Role: Sends request to Runway API to create video from images and scripts.  
  - Inputs: From BackgroundType node (true branch).  
  - Outputs: To Wait2 and Execute Workflow2 nodes.

- **Runway - Get Video**  
  - Type: HTTP Request  
  - Role: Polls Runway API for video generation status and retrieves video URL.  
  - Inputs: From Wait2 node.  
  - Outputs: To output node and Execute Workflow6 sub-workflow.

- **Wait2**  
  - Type: Wait  
  - Role: Pauses workflow to allow Runway video processing.  
  - Inputs: From Runway - Create Video node.  
  - Outputs: To Runway - Get Video.

- **Execute Workflow2** and **Execute Workflow6**  
  - Type: Execute Workflow  
  - Role: Sub-workflows for processing Runway video creation and post-processing.  
  - Inputs: From Runway - Create Video and Runway - Get Video respectively.

- **HeyGen**  
  - Type: HTTP Request  
  - Role: Sends requests to HeyGen API for avatar video creation.  
  - Inputs: From If_with_heygen node.  
  - Outputs: To Wait4 node and heygen Execute ERROR node.

- **HeyGen : Check Video**  
  - Type: HTTP Request  
  - Role: Polls HeyGen API to check if video rendering is complete.  
  - Inputs: From Wait4 node.  
  - Outputs: To heygen_response switch node and heygen Execute ERROR2 node.

- **Wait4**  
  - Type: Wait  
  - Role: Waits for HeyGen video generation completion asynchronously.  
  - Inputs: From HeyGen node.  
  - Outputs: To HeyGen : Check Video node.

- **heygen_response**  
  - Type: Switch  
  - Role: Branches based on HeyGen video check response status.  
  - Inputs: From HeyGen : Check Video node.  
  - Outputs:  
    - To Code Heygen node (successful).  
    - To Wait4 node (wait and retry).

- **Code Heygen**  
  - Type: Code  
  - Role: Processes HeyGen response data, possibly extracting video URL or metadata.  
  - Inputs: From heygen_response switch node.  
  - Outputs: To json2video : Video Rendering node.

- **json2video : Video Rendering**  
  - Type: HTTP Request  
  - Role: Initiates video rendering on json2video platform.  
  - Inputs: From Code Heygen and Code Add Sub nodes.  
  - Outputs: To Wait node and json2video Execute ERROR node.

- **json2video : Check Video Rendering**  
  - Type: HTTP Request  
  - Role: Polls json2video API to check video rendering status.  
  - Inputs: From Wait node.  
  - Outputs: To j2v_response switch node.

- **Wait**  
  - Type: Wait  
  - Role: Waits between polling attempts for json2video rendering.  
  - Inputs: From json2video : Video Rendering node.  
  - Outputs: To json2video : Check Video Rendering.

- **j2v_response**  
  - Type: Switch  
  - Role: Branches based on json2video rendering status.  
  - Inputs: From json2video : Check Video Rendering node.  
  - Outputs:  
    - To Baserow node (success).  
    - To Wait node (retry).  
    - To json2video Execute ERROR1 node (failure).

- **Error Execution Workflows**  
  - **json2video Execute ERROR**  
  - **json2video Execute ERROR1**  
  - **heygen Execute ERROR**  
  - **heygen Execute ERROR2**  
  - Type: Execute Workflow  
  - Role: Handle errors encountered during video creation or polling phases.  
  - Inputs: From respective error branches in main flow.

**Edge Cases / Failures:**  
- API rate limits or auth errors with Runway, HeyGen, json2video.  
- Polling timeouts or webhook failures in wait nodes.  
- Unexpected API responses causing switch branching issues.  
- Sub-workflow errors during video post-processing.

---

### 1.5 Caption Generation

**Overview:**  
Generates video captions using CaptionsAI API, polls for completion, aggregates caption data, and handles errors.

**Nodes Involved:**  
- CaptionsAI1 (HTTP Request)  
- CaptionsAI : Check Poll1 (HTTP Request)  
- Wait6 (Wait)  
- cap_response (Switch)  
- Aggregate  
- CAPTIONS Execute ERROR  
- CAPTIONS Execute ERROR1

**Node Details:**

- **CaptionsAI1**  
  - Type: HTTP Request  
  - Role: Initiates caption generation job with CaptionsAI API.  
  - Inputs: From Aggregate node.  
  - Outputs: To Wait6 node and CAPTIONS Execute ERROR1 node.

- **CaptionsAI : Check Poll1**  
  - Type: HTTP Request  
  - Role: Polls CaptionsAI API to check status of caption generation.  
  - Inputs: From Wait6 node.  
  - Outputs: To cap_response switch node.

- **Wait6**  
  - Type: Wait  
  - Role: Waits between polling attempts for caption generation completion.  
  - Inputs: From CaptionsAI1 node.  
  - Outputs: To CaptionsAI : Check Poll1 node.

- **cap_response**  
  - Type: Switch  
  - Role: Branches based on caption generation status.  
  - Inputs: From CaptionsAI : Check Poll1 node.  
  - Outputs:  
    - To Code Add Sub node (success).  
    - To Wait6 node (retry).  
    - To CAPTIONS Execute ERROR node (failure).

- **Aggregate**  
  - Type: Aggregate  
  - Role: Aggregates data from previous steps to pass to CaptionsAI1 node.  
  - Inputs: From If_with_avatar and loop_over_scenes logic.  
  - Outputs: To CaptionsAI1 node.

- **CAPTIONS Execute ERROR & CAPTIONS Execute ERROR1**  
  - Type: Execute Workflow  
  - Role: Sub-workflows to handle caption-related errors during initiation or polling.  
  - Inputs: From respective error branches.

**Edge Cases / Failures:**  
- API errors or rate limits with CaptionsAI.  
- Polling timeouts or webhook failures in Wait6 node.  
- Logic errors in status switching.  
- Sub-workflow error handling.

---

### 1.6 Error Handling & Workflow Control

**Overview:**  
Handles errors by logging to Baserow, stopping execution, and managing workflow triggers.

**Nodes Involved:**  
- Baserow Error (Baserow)  
- Stop and Error (Stop and Error)  
- When Executed by Another Workflow (Execute Workflow Trigger)

**Node Details:**

- **Baserow Error**  
  - Type: Baserow  
  - Role: Logs error messages or statuses in the Baserow database.  
  - Inputs: From When Executed by Another Workflow node and other error branches.  
  - Outputs: To Stop and Error node.

- **Stop and Error**  
  - Type: Stop and Error  
  - Role: Terminates workflow execution with error message.  
  - Inputs: From Baserow Error node.  
  - Outputs: None (workflow ends).

- **When Executed by Another Workflow**  
  - Type: Execute Workflow Trigger  
  - Role: Listens for external workflow triggers to start error handling or other processes.  
  - Inputs: External triggers.  
  - Outputs: To Baserow Error node.

**Edge Cases / Failures:**  
- Database connectivity errors.  
- Unexpected termination conditions.  
- Trigger misconfiguration.

---

### 1.7 Sub-Workflow Executions

**Overview:**  
Delegates complex or reusable tasks to sub-workflows for modularity and maintainability.

**Nodes Involved:**  
- Execute Workflow  
- Execute Workflow2  
- Execute Workflow3  
- Execute Workflow4  
- Execute Workflow5  
- Execute Workflow6  
- json2video Execute ERROR  
- heygen Execute ERROR  
- heygen Execute ERROR2  
- CAPTIONS Execute ERROR  
- CAPTIONS Execute ERROR1  
- json2video Execute ERROR1

**Node Details:**

- **Execute Workflow Nodes**  
  - Type: Execute Workflow  
  - Role: Trigger named sub-workflows with relevant input data.  
  - Configuration: Each points to a specific sub-workflow responsible for tasks like script processing, image processing, error handling, or video post-processing.  
  - Inputs: From various points in the main workflow.  
  - Outputs: None explicitly; may affect main workflow via callbacks or data updates.

**Version Requirements:** Sub-workflows must be compatible with n8n version used.

**Edge Cases / Failures:**  
- Sub-workflow missing or misconfiguration.  
- Credential or parameter mismatches.  
- Error propagation from sub-workflows.

---

## 3. Summary Table

| Node Name                     | Node Type                         | Functional Role                                       | Input Node(s)                         | Output Node(s)                          | Sticky Note |
|-------------------------------|----------------------------------|-----------------------------------------------------|-------------------------------------|----------------------------------------|-------------|
| Webhook                       | Webhook (Trigger)                 | Entry point; receives incoming requests             | -                                   | Body                                   |             |
| Body                         | Set                              | Sets/formats incoming request data                   | Webhook                             | Should Process?                        |             |
| Should Process?               | If                               | Decision to process or not                            | Body                                | Switch ScriptType, Baserow Processing  |             |
| Switch ScriptType             | Switch                           | Routes based on script type                           | Should Process? (true)               | If, Basic LLM Chain                    |             |
| If                           | If                               | Branches manual vs automatic script processing       | Switch ScriptType                   | Basic LLM Chain Manual, Execute Workflow |             |
| Basic LLM Chain              | LangChain LLM Chain              | Automatic script generation and processing           | Switch ScriptType, If                | Scenes Mapping, Update Script, Execute Workflow4 |             |
| Basic LLM Chain Manual       | LangChain LLM Chain              | Manual script improvement                             | If                                 | Update Script, Scenes Mapping, Execute Workflow4 |             |
| OpenAI Chat Model            | LangChain AI Language Model      | AI model for script generation                        | -                                  | Basic LLM Chain                        |             |
| OpenAI Chat Model1           | LangChain AI Language Model      | AI model for manual script processing                 | -                                  | Basic LLM Chain Manual                 |             |
| Structured Output Parser     | LangChain Output Parser          | Parses AI responses into structured data             | Basic LLM Chain, Basic LLM Chain Manual | Basic LLM Chain, Basic LLM Chain Manual |             |
| Update Script                | Baserow                         | Updates script records in database                     | Basic LLM Chain, Basic LLM Chain Manual | -                                  |             |
| Execute Workflow4            | Execute Workflow                 | Sub-workflow for script/scene processing              | Basic LLM Chain, Basic LLM Chain Manual | -                                  |             |
| Execute Workflow             | Execute Workflow                 | Alternative sub-workflow for manual scripts           | If                                 | -                                      |             |
| Scenes Mapping               | Set                              | Formats scenes for processing                          | Basic LLM Chain                    | Split Out                             |             |
| Split Out                   | SplitOut                        | Splits scenes array into individual scene items       | Scenes Mapping                    | loop_over_scenes                     |             |
| loop_over_scenes             | SplitInBatches                  | Batches scene processing                               | output, Split Out                  | If_with_avatar, Leo - Improve Prompt |             |
| Leo - Improve Prompt         | HTTP Request                   | Improves image generation prompts                      | loop_over_scenes                  | Leo - Generate Image, Execute Workflow5 |             |
| Leo - Generate Image         | HTTP Request                   | Requests image generation                              | Leo - Improve Prompt              | Wait1, Execute Workflow3              |             |
| Wait1                       | Wait                           | Waits for asynchronous image generation               | Leo - Generate Image              | Leo - Get imageId                    |             |
| Leo - Get imageId            | HTTP Request                   | Retrieves generated image ID                           | Wait1                            | BackgroundType                      |             |
| BackgroundType               | If                             | Decides video creation path based on background type | Leo - Get imageId                 | Runway - Create Video, output image  |             |
| output image                | Set                            | Sets image data for further processing                 | BackgroundType                   | loop_over_scenes                   |             |
| Execute Workflow3            | Execute Workflow               | Sub-workflow for image processing                      | Leo - Generate Image              | -                                  |             |
| Execute Workflow5            | Execute Workflow               | Sub-workflow for prompt improvement                    | Leo - Improve Prompt              | -                                  |             |
| Runway - Create Video        | HTTP Request                   | Creates video on Runway platform                        | BackgroundType (true branch)      | Wait2, Execute Workflow2             |             |
| Wait2                       | Wait                           | Waits for Runway video processing                       | Runway - Create Video             | Runway - Get Video                  |             |
| Runway - Get Video           | HTTP Request                   | Polls Runway for video completion and retrieval        | Wait2                            | output, Execute Workflow6            |             |
| Execute Workflow2            | Execute Workflow               | Sub-workflow for Runway video processing                | Runway - Create Video             | -                                  |             |
| Execute Workflow6            | Execute Workflow               | Sub-workflow for Runway video post-processing           | Runway - Get Video                | -                                  |             |
| HeyGen                      | HTTP Request                   | Requests HeyGen avatar video creation                   | If_with_heygen                   | Wait4, heygen Execute ERROR          |             |
| Wait4                       | Wait                           | Waits for HeyGen video rendering                        | HeyGen                          | HeyGen : Check Video                |             |
| HeyGen : Check Video         | HTTP Request                   | Polls HeyGen for video rendering status                 | Wait4                           | heygen_response, heygen Execute ERROR2 |             |
| heygen_response             | Switch                         | Branches based on HeyGen video status                   | HeyGen : Check Video             | Code Heygen, Wait4                  |             |
| Code Heygen                 | Code                           | Processes HeyGen response data                           | heygen_response                 | json2video : Video Rendering       |             |
| json2video : Video Rendering | HTTP Request                   | Initiates video rendering on json2video platform        | Code Heygen, Code Add Sub        | Wait, json2video Execute ERROR     |             |
| Wait                        | Wait                           | Waits between json2video rendering status polls         | json2video : Video Rendering     | json2video : Check Video Rendering |             |
| json2video : Check Video Rendering | HTTP Request             | Polls json2video for rendering status                    | Wait                           | j2v_response                      |             |
| j2v_response                | Switch                         | Branches on json2video rendering status                  | json2video : Check Video Rendering | Baserow, Wait, json2video Execute ERROR1 |             |
| Baserow                     | Baserow                       | Updates database with video info                          | j2v_response                   | Wait                             |             |
| CaptionsAI1                 | HTTP Request                   | Starts caption generation job                            | Aggregate                     | Wait6, CAPTIONS Execute ERROR1     |             |
| Wait6                       | Wait                           | Waits between caption polling                            | CaptionsAI1                   | CaptionsAI : Check Poll1           |             |
| CaptionsAI : Check Poll1     | HTTP Request                   | Polls caption generation status                           | Wait6                         | cap_response                     |             |
| cap_response                | Switch                         | Branches on caption generation status                    | CaptionsAI : Check Poll1       | Code Add Sub, Wait6, CAPTIONS Execute ERROR |             |
| Aggregate                   | Aggregate                     | Aggregates data for captions API                          | If_with_avatar, loop_over_scenes | CaptionsAI1                     |             |
| Code Add Sub                | Code                           | Processes caption data                                    | cap_response                  | json2video : Video Rendering      |             |
| CAPTIONS Execute ERROR      | Execute Workflow               | Handles caption generation errors                         | cap_response                  | -                              |             |
| CAPTIONS Execute ERROR1     | Execute Workflow               | Handles caption initiation errors                         | CaptionsAI1                   | -                              |             |
| Baserow Error               | Baserow                       | Logs errors in database                                   | When Executed by Another Workflow | Stop and Error                  |             |
| Stop and Error              | Stop and Error                | Terminates workflow on error                              | Baserow Error                 | -                              |             |
| When Executed by Another Workflow | Execute Workflow Trigger | Trigger for error handling workflows                      | External trigger              | Baserow Error                   |             |
| Execute Workflow            | Execute Workflow               | Sub-workflow for manual script processing                 | If                           | -                              |             |
| Execute Workflow3           | Execute Workflow               | Sub-workflow for image processing                         | Leo - Generate Image          | -                              |             |
| Execute Workflow4           | Execute Workflow               | Sub-workflow for scene/script processing                  | Basic LLM Chain, Basic LLM Chain Manual | -                      |             |
| Execute Workflow5           | Execute Workflow               | Sub-workflow for prompt improvement                       | Leo - Improve Prompt          | -                              |             |
| Execute Workflow6           | Execute Workflow               | Sub-workflow for Runway video post-processing             | Runway - Get Video            | -                              |             |
| json2video Execute ERROR    | Execute Workflow               | Handles errors in json2video video rendering              | json2video : Video Rendering  | -                              |             |
| json2video Execute ERROR1   | Execute Workflow               | Handles json2video rendering error during polling         | j2v_response                 | -                              |             |
| heygen Execute ERROR        | Execute Workflow               | Handles HeyGen API errors                                  | HeyGen                      | -                              |             |
| heygen Execute ERROR2       | Execute Workflow               | Handles HeyGen video checking errors                      | HeyGen : Check Video         | -                              |             |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Configure HTTP method and path to receive incoming video creation requests.

2. **Add Set Node ("Body")**  
   - Use to extract and format webhook request body data for downstream nodes.

3. **Add If Node ("Should Process?")**  
   - Configure conditions to determine if the incoming request should be processed (e.g., validate fields).

4. **Add Switch Node ("Switch ScriptType")**  
   - Route based on script type field (manual vs automatic).

5. **Add If Node ("If")**  
   - Branch workflow for manual script processing vs automatic.

6. **Add LangChain Nodes for Script Generation**  
   - Add two **Basic LLM Chain** nodes (one for automatic, one manual).  
   - Attach **OpenAI Chat Model** nodes as AI language models to these chains.  
   - Add **Structured Output Parser** node connected to both LLM chains for output parsing.

7. **Add Baserow Node ("Update Script")**  
   - Configure to update or insert script data based on AI-generated content.

8. **Add Execute Workflow Nodes ("Execute Workflow4" and "Execute Workflow")**  
   - Set up sub-workflows for further script or manual processing.

9. **Add Set Node ("Scenes Mapping")**  
   - Map scenes extracted from script for image generation.

10. **Add SplitOut Node ("Split Out")**  
    - Split scenes array into individual items.

11. **Add SplitInBatches Node ("loop_over_scenes")**  
    - Process scenes in batches.

12. **Add If Node ("If_with_avatar")**  
    - Check if avatar processing is needed.

13. **Add Aggregate Node ("Aggregate")**  
    - Aggregate scene data for captioning.

14. **Add HTTP Request Nodes for Image Generation via Leo API**  
    - "Leo - Improve Prompt": Refine image prompts.  
    - "Leo - Generate Image": Request image creation.  
    - "Leo - Get imageId": Retrieve generated image ID.

15. **Add Wait Node ("Wait1")**  
    - Wait for asynchronous image generation.

16. **Add If Node ("BackgroundType")**  
    - Determine if Runway video creation or output image flow is needed.

17. **Add HTTP Request Nodes for Runway Video Creation**  
    - "Runway - Create Video": Request video creation.  
    - "Runway - Get Video": Poll for video readiness.

18. **Add Wait Node ("Wait2")**  
    - Wait between Runway API polls.

19. **Add Execute Workflow Nodes ("Execute Workflow2", "Execute Workflow3", "Execute Workflow5", "Execute Workflow6")**  
    - Sub-workflows for image and video processing.

20. **Add HTTP Request Nodes for HeyGen Avatar Video Creation**  
    - "HeyGen": Request avatar video creation.  
    - "HeyGen : Check Video": Poll video status.

21. **Add Wait Node ("Wait4")**  
    - Wait for HeyGen video completion.

22. **Add Switch Node ("heygen_response")**  
    - Branch on HeyGen video status.

23. **Add Code Node ("Code Heygen")**  
    - Process HeyGen API response.

24. **Add HTTP Request Nodes for json2video Platform**  
    - "json2video : Video Rendering": Start rendering.  
    - "json2video : Check Video Rendering": Poll rendering status.

25. **Add Wait Node ("Wait")**  
    - Wait between json2video polls.

26. **Add Switch Node ("j2v_response")**  
    - Branch on json2video rendering status.

27. **Add Baserow Node ("Baserow")**  
    - Update video records.

28. **Add HTTP Request Nodes for Captions Generation**  
    - "CaptionsAI1": Start captions job.  
    - "CaptionsAI : Check Poll1": Poll captions status.

29. **Add Wait Node ("Wait6")**  
    - Wait between captions API polls.

30. **Add Switch Node ("cap_response")**  
    - Branch on captions generation status.

31. **Add Code Node ("Code Add Sub")**  
    - Process captions data for downstream video rendering.

32. **Add Execute Workflow Nodes for Error Handling**  
    - "json2video Execute ERROR", "json2video Execute ERROR1", "heygen Execute ERROR", "heygen Execute ERROR2", "CAPTIONS Execute ERROR", "CAPTIONS Execute ERROR1".

33. **Add Baserow Error Logging Node ("Baserow Error")**  
    - Logs errors to database.

34. **Add Stop and Error Node**  
    - Terminates workflow on errors.

35. **Add Execute Workflow Trigger Node ("When Executed by Another Workflow")**  
    - Listens for external workflow triggers to start error handling.

36. **Connect all nodes according to flow described in section 2.**  
    - Ensure proper data passing and error flow.

37. **Configure Credentials**  
    - OpenAI API credentials for language models.  
    - HTTP credentials/tokens for Leo, Runway, HeyGen, json2video, CaptionsAI APIs.  
    - Baserow API credentials.

38. **Set Default Values and Parameters**  
    - Configure polling intervals, timeouts, field mappings.  
    - Set batch size for SplitInBatches node.  
    - Define conditions in If and Switch nodes based on expected data schema.

39. **Deploy and Test**  
    - Perform test calls to webhook with sample payloads.  
    - Verify each branch and error path.

---

## 5. General Notes & Resources

| Note Content                                                                                                                     | Context or Link                                                                                                                                                      |
|----------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| This workflow uses AI models and external APIs extensively; ensure all API keys are securely stored and access is configured.  | Credential management in n8n.                                                                                                                                        |
| HeyGen avatar video creation is a limited-time offer feature as indicated in the workflow title.                                | https://heygen.com/ (for avatar video generation)                                                                                                                  |
| Baserow is used as a low-code database solution to manage scripts, videos, and error logs.                                       | https://baserow.io                                                                                                                                                   |
| The workflow relies on asynchronous video/image generation; careful timeout and retry management is critical for reliability.   | n8n Wait node documentation: https://docs.n8n.io/nodes/n8n-nodes-base.wait/                                                                                        |
| Sub-workflows modularize complex tasks, improving maintainability and scalability.                                               | For sub-workflow design best practices see: https://docs.n8n.io/workflows/sub-workflows/                                                                            |
| Polling mechanisms are used for checking video rendering status on Runway, HeyGen, and json2video platforms.                     | Polling pattern implementation reference: https://docs.n8n.io/nodes/n8n-nodes-base.wait/                                                                           |
| Error handling via dedicated error workflows ensures separation of concerns and better debugging.                               | n8n error workflow guide: https://docs.n8n.io/workflows/error-handling/                                                                                            |

---

*Disclaimer: The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.*
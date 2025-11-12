Place People in Iconic Locations using Input Images, Flux Kontext and Replicate

https://n8nworkflows.xyz/workflows/place-people-in-iconic-locations-using-input-images--flux-kontext-and-replicate-6871


# Place People in Iconic Locations using Input Images, Flux Kontext and Replicate

### 1. Workflow Overview

This workflow automates the generation of images that place people into iconic locations using input images. It leverages the Flux Kontext "iconic-locations" AI model hosted on the Replicate API. The workflow is designed for users who want to transform a portrait or other image into a creative output where the subject appears in a famous landmark or recognizable place.

The workflow is logically divided into these functional blocks:

- **1.1 Trigger and Initialization**: Starts the workflow manually and sets up the API authentication token.
- **1.2 Parameter Setup**: Defines and configures all the input parameters required for the image generation model, including image URL, location, and generation options.
- **1.3 Prediction Request and Monitoring**: Sends a generation request to the Replicate API and continuously polls the status until the image is ready or a failure occurs.
- **1.4 Response Handling**: Processes successful or failed generation attempts, formats output responses, and logs request details.
- **1.5 Logging and Monitoring**: Captures and logs key request details for debugging and auditing.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and Initialization

**Overview:**  
This block starts the workflow on manual user trigger and sets the Replicate API token for authentication.

**Nodes Involved:**  
- Manual Trigger  
- Set API Token

**Node Details:**  

- **Manual Trigger**  
  - Type: `manualTrigger`  
  - Role: Initiates the workflow manually.  
  - Configuration: Default, no parameters.  
  - Inputs: None  
  - Outputs: Connects to "Set API Token"  
  - Failure Modes: None, as execution is manual.  

- **Set API Token**  
  - Type: `set`  
  - Role: Stores the Replicate API token in the workflow context.  
  - Configuration: Sets a string variable `api_token` with placeholder `YOUR_REPLICATE_API_TOKEN` which must be replaced with a valid token.  
  - Inputs: From "Manual Trigger"  
  - Outputs: Connects to "Set Image Parameters"  
  - Failure Modes: Missing or invalid API token will cause authentication errors downstream.  

---

#### 2.2 Parameter Setup

**Overview:**  
Prepares all input parameters for the image generation model, including required and optional fields with sensible defaults.

**Nodes Involved:**  
- Set Image Parameters

**Node Details:**  

- **Set Image Parameters**  
  - Type: `set`  
  - Role: Defines and passes model input parameters such as seed, gender, input image URL, aspect ratio, output format, iconic location, and safety tolerance.  
  - Configuration:  
    - Copies `api_token` from previous node.  
    - `seed`: -1 (random seed)  
    - `gender`: "none"  
    - `input_image`: URL "https://picsum.photos/512/512" as a default placeholder.  
    - `aspect_ratio`: "match_input_image"  
    - `output_format`: "png"  
    - `iconic_location`: "Eiffel Tower"  
    - `safety_tolerance`: 2 (most permissive)  
  - Inputs: From "Set API Token"  
  - Outputs: Connects to "Create Image Prediction"  
  - Failure Modes: Invalid parameter values (e.g., unsupported image URL or wrong aspect ratio) may cause API errors.

---

#### 2.3 Prediction Request and Monitoring

**Overview:**  
Sends the image generation request to the Replicate API and polls the prediction status until it either succeeds or fails, implementing wait periods between status checks.

**Nodes Involved:**  
- Create Image Prediction  
- Log Request  
- Wait 5s  
- Check Status  
- Is Complete?  
- Has Failed?  
- Wait 10s

**Node Details:**  

- **Create Image Prediction**  
  - Type: `httpRequest` (POST)  
  - Role: Sends the generation request to Replicate API endpoint `https://api.replicate.com/v1/predictions` with all input parameters.  
  - Configuration:  
    - JSON body includes model version and input parameters dynamically mapped from previous node.  
    - Headers include Authorization Bearer token and `Prefer: wait` to wait for initial response.  
    - Always accepts response without error to handle status checking manually.  
  - Inputs: From "Set Image Parameters"  
  - Outputs: Connects to "Log Request"  
  - Failure Modes: HTTP errors, API rate limits, or invalid token can cause failures.

- **Log Request**  
  - Type: `code`  
  - Role: Logs key prediction details (timestamp, prediction ID, model type) to console for monitoring.  
  - Configuration: Custom JavaScript code extracting prediction ID and logging it.  
  - Inputs: From "Create Image Prediction"  
  - Outputs: Connects to "Wait 5s"  
  - Failure Modes: None significant, logging errors won't stop workflow.

- **Wait 5s**  
  - Type: `wait`  
  - Role: Pauses workflow for 5 seconds before checking status.  
  - Inputs: From "Log Request"  
  - Outputs: Connects to "Check Status"  
  - Failure Modes: None.

- **Check Status**  
  - Type: `httpRequest` (GET)  
  - Role: Queries Replicate API prediction status using prediction ID.  
  - Configuration:  
    - URL dynamically constructed using prediction ID from "Create Image Prediction".  
    - Authorization header with API token.  
    - Allows no error on response to handle different states manually.  
  - Inputs: From "Wait 5s" or "Wait 10s"  
  - Outputs: Connects to "Is Complete?"  
  - Failure Modes: Network issues or invalid ID cause request failures.

- **Is Complete?**  
  - Type: `if`  
  - Role: Checks if the prediction status is `"succeeded"`.  
  - Configuration: Condition compares `$json.status` to `"succeeded"`.  
  - Inputs: From "Check Status"  
  - Outputs:  
    - True branch: "Success Response"  
    - False branch: "Has Failed?"  
  - Failure Modes: Expression errors if status field missing.

- **Has Failed?**  
  - Type: `if`  
  - Role: Checks if the prediction status is `"failed"`.  
  - Configuration: Condition compares `$json.status` to `"failed"`.  
  - Inputs: From "Is Complete?" (False branch)  
  - Outputs:  
    - True branch: "Error Response"  
    - False branch: "Wait 10s" (retry)  
  - Failure Modes: Expression errors if status missing.

- **Wait 10s**  
  - Type: `wait`  
  - Role: Waits 10 seconds before retrying status check.  
  - Inputs: From "Has Failed?" (False branch)  
  - Outputs: Connects back to "Check Status" to loop polling.  
  - Failure Modes: None.

---

#### 2.4 Response Handling

**Overview:**  
Processes the final output after prediction completes successfully or fails, formats structured JSON responses for downstream consumption.

**Nodes Involved:**  
- Success Response  
- Error Response  
- Display Result

**Node Details:**  

- **Success Response**  
  - Type: `set`  
  - Role: Constructs a JSON response object with success status, image URL, prediction ID, status, and message.  
  - Configuration: Sets `response` object with keys: `success:true`, `image_url`, `prediction_id`, `status`, `message`. Values extracted from prediction JSON output.  
  - Inputs: From "Is Complete?" (True branch)  
  - Outputs: Connects to "Display Result"  
  - Failure Modes: Missing output fields may cause incomplete response.

- **Error Response**  
  - Type: `set`  
  - Role: Constructs a JSON response object for failed generation including error details and message.  
  - Configuration: Sets `response` object with keys: `success:false`, `error`, `prediction_id`, `status`, `message`. Error message defaults if none provided.  
  - Inputs: From "Has Failed?" (True branch)  
  - Outputs: Connects to "Display Result"  
  - Failure Modes: Missing error field handled with default message.

- **Display Result**  
  - Type: `set`  
  - Role: Moves the `response` object to a `final_result` field for output or further use.  
  - Configuration: Sets `final_result` equal to the `response` object from either success or error branch.  
  - Inputs: From "Success Response" and "Error Response"  
  - Outputs: End of workflow  
  - Failure Modes: None.

---

#### 2.5 Logging and Monitoring

**Overview:**  
Captures logs of all requests including timestamps and prediction IDs for auditing and debugging.

**Nodes Involved:**  
- Log Request (also part of Prediction Request block)

**Node Details:**  

- **Log Request** (see above)  
  - Logs detailed request info including timestamp and model type.  
  - No external logging service connected; logs appear in n8n console.

---

### 3. Summary Table

| Node Name            | Node Type        | Functional Role                                  | Input Node(s)        | Output Node(s)           | Sticky Note                                                                                                   |
|----------------------|------------------|-------------------------------------------------|----------------------|--------------------------|---------------------------------------------------------------------------------------------------------------|
| Manual Trigger       | manualTrigger    | Starts workflow manually                         | None                 | Set API Token             | =======================================<br>ICONIC-LOCATIONS GENERATOR<br>For support contact Yaron@nofluff.online<br>See YouTube and LinkedIn links in note |
| Set API Token        | set              | Sets Replicate API token variable                | Manual Trigger       | Set Image Parameters      | See Sticky Note4 for detailed workflow explanation and parameters                                            |
| Set Image Parameters | set              | Configures all input parameters for image model | Set API Token        | Create Image Prediction   | See Sticky Note4 for detailed workflow explanation and parameters                                            |
| Create Image Prediction | httpRequest    | Sends generation request to Replicate API        | Set Image Parameters | Log Request               | See Sticky Note4                                                                                              |
| Log Request          | code             | Logs request details for monitoring               | Create Image Prediction | Wait 5s                 | See Sticky Note4                                                                                              |
| Wait 5s              | wait             | Waits 5 seconds before status check               | Log Request          | Check Status              | See Sticky Note4                                                                                              |
| Check Status         | httpRequest      | Checks prediction status from API                 | Wait 5s, Wait 10s    | Is Complete?              | See Sticky Note4                                                                                              |
| Is Complete?         | if               | Checks if prediction succeeded                     | Check Status         | Success Response, Has Failed? | See Sticky Note4                                                                                              |
| Has Failed?          | if               | Checks if prediction failed                        | Is Complete? (False) | Error Response, Wait 10s  | See Sticky Note4                                                                                              |
| Wait 10s             | wait             | Waits 10 seconds before retrying status check     | Has Failed? (False)  | Check Status              | See Sticky Note4                                                                                              |
| Success Response     | set              | Formats success JSON response                       | Is Complete? (True)  | Display Result            | See Sticky Note4                                                                                              |
| Error Response       | set              | Formats error JSON response                         | Has Failed? (True)   | Display Result            | See Sticky Note4                                                                                              |
| Display Result       | set              | Sets final output result                            | Success Response, Error Response | None                    | See Sticky Note4                                                                                              |
| Sticky Note9         | stickyNote       | Branding, contact info, and support links          | None                 | None                     | =======================================<br>ICONIC-LOCATIONS GENERATOR<br>Contact and social links provided              |
| Sticky Note4         | stickyNote       | Detailed workflow overview, parameter guide, and instructions | None          | None                     | Extensive documentation within the workflow, includes links to docs and troubleshooting                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Manual Trigger node:**  
   - Name: `Manual Trigger`  
   - No configuration needed.  
   - This node will start the workflow manually.

3. **Add a Set node for API token:**  
   - Name: `Set API Token`  
   - Add a string parameter: `api_token`  
   - Set value to your Replicate API token (replace `"YOUR_REPLICATE_API_TOKEN"`).  
   - Connect `Manual Trigger` output to this node.

4. **Add a Set node for image parameters:**  
   - Name: `Set Image Parameters`  
   - Add parameters with these keys and default values:  
     - `api_token` (string): Expression `{{$node["Set API Token"].json["api_token"]}}`  
     - `seed` (number): `-1`  
     - `gender` (string): `"none"`  
     - `input_image` (string): `"https://picsum.photos/512/512"`  
     - `aspect_ratio` (string): `"match_input_image"`  
     - `output_format` (string): `"png"`  
     - `iconic_location` (string): `"Eiffel Tower"`  
     - `safety_tolerance` (number): `2`  
   - Connect `Set API Token` output to this node.

5. **Add an HTTP Request node to create image prediction:**  
   - Name: `Create Image Prediction`  
   - Set method to POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Add headers:  
     - `Authorization`: `Bearer {{$json.api_token}}`  
     - `Prefer`: `wait`  
   - Body Content Type: JSON  
   - JSON Body:  
     ```json
     {
       "version": "flux-kontext-apps/iconic-locations:cff3e2c056e4f9c0b829dc290037e7b0ceec3a294ecd6c246136aeff334585c1",
       "input": {
         "seed": {{$json.seed}},
         "gender": "{{$json.gender}}",
         "input_image": "{{$json.input_image}}",
         "aspect_ratio": "{{$json.aspect_ratio}}",
         "output_format": "{{$json.output_format}}",
         "iconic_location": "{{$json.iconic_location}}",
         "safety_tolerance": {{$json.safety_tolerance}}
       }
     }
     ```  
   - Connect `Set Image Parameters` output to this node.

6. **Add a Code node for logging request details:**  
   - Name: `Log Request`  
   - Code (JavaScript):  
     ```js
     const data = $input.all()[0].json;
     console.log('flux-kontext-apps/iconic-locations Request:', {
       timestamp: new Date().toISOString(),
       prediction_id: data.id,
       model_type: 'image'
     });
     return $input.all();
     ```  
   - Connect `Create Image Prediction` output to this node.

7. **Add a Wait node:**  
   - Name: `Wait 5s`  
   - Set wait time to 5 seconds.  
   - Connect `Log Request` output to this node.

8. **Add an HTTP Request node to check status:**  
   - Name: `Check Status`  
   - Method: GET  
   - URL: `https://api.replicate.com/v1/predictions/{{$node["Create Image Prediction"].json["id"]}}`  
   - Headers:  
     - `Authorization`: `Bearer {{$node["Set API Token"].json["api_token"]}}`  
   - Allow no error on response to handle manual status checks.  
   - Connect `Wait 5s` output to this node.  
   - Connect also from a later retry Wait node (see below).

9. **Add an If node to check if prediction succeeded:**  
   - Name: `Is Complete?`  
   - Condition: Check if `$json.status` equals `"succeeded"`  
   - Connect `Check Status` output to this node.

10. **Add an If node to check if prediction failed:**  
    - Name: `Has Failed?`  
    - Condition: Check if `$json.status` equals `"failed"`  
    - Connect `Is Complete?` (False branch) to this node.

11. **Add a Set node for success response:**  
    - Name: `Success Response`  
    - Set a field `response` with object:  
      ```json
      {
        "success": true,
        "image_url": "{{$json.output}}",
        "prediction_id": "{{$json.id}}",
        "status": "{{$json.status}}",
        "message": "Image generated successfully"
      }
      ```  
    - Connect `Is Complete?` (True branch) to this node.

12. **Add a Set node for error response:**  
    - Name: `Error Response`  
    - Set a field `response` with object:  
      ```json
      {
        "success": false,
        "error": "{{$json.error || 'Image generation failed'}}",
        "prediction_id": "{{$json.id}}",
        "status": "{{$json.status}}",
        "message": "Failed to generate image"
      }
      ```  
    - Connect `Has Failed?` (True branch) to this node.

13. **Add a Wait node for retry delay:**  
    - Name: `Wait 10s`  
    - Set wait time to 10 seconds.  
    - Connect `Has Failed?` (False branch) to this node.  
    - Connect `Wait 10s` output back to `Check Status` to form polling loop.

14. **Add a Set node to finalize output:**  
    - Name: `Display Result`  
    - Set a field `final_result` equal to `{{$json.response}}`  
    - Connect both `Success Response` and `Error Response` outputs to this node.

15. **Test workflow by triggering Manual Trigger.**  
    - Replace the API token before use.  
    - Customize input parameters in `Set Image Parameters` as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Context or Link                                                                                     |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| =======================================<br>ICONIC-LOCATIONS GENERATOR<br>For any questions or support, please contact:<br>Yaron@nofluff.online<br>Explore more tips and tutorials here:<br>- YouTube: https://www.youtube.com/@YaronBeen/videos<br>- LinkedIn: https://www.linkedin.com/in/yaronbeen/<br>=======================================                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Provided as Sticky Note9 in workflow for branding and support contacts                            |
| ## 🤖 **FLUX-KONTEXT-APPS/ICONIC-LOCATIONS - IMAGE GENERATION WORKFLOW**<br>**🔥 Powered by Replicate API and n8n Automation**<br>---<br>### 📝 **Model Overview**<br>- **Owner**: flux-kontext-apps<br>- **Model**: iconic-locations<br>- **Type**: Image Generation<br>- **API Endpoint**: https://api.replicate.com/v1/predictions<br>**🎯 What This Model Does:**<br>Put yourself in an iconic location around the world from a single image<br>---<br>### 📋 **Parameter Reference**<br>**🔴 Required Parameters:** input_image<br>**🔵 Optional Parameters:** seed, gender, aspect_ratio, output_format, iconic_location, safety_tolerance<br>---<br>### 🔧 **Workflow Components Explained**<br>Detailed explanation of nodes and logic.<br>---<br>### 🔍 **Troubleshooting Guide**<br>Common issues and best practices.<br>---<br>**🔗 Additional Resources:**<br>- Model Documentation: https://replicate.com/flux-kontext-apps/iconic-locations<br>- Replicate API Docs: https://replicate.com/docs<br>- n8n Documentation: https://docs.n8n.io<br>--- | Sticky Note4 contains comprehensive workflow documentation, parameter guide, and troubleshooting. |

---

This documentation fully describes the "Place People in Iconic Locations using Input Images, Flux Kontext and Replicate" workflow, enabling advanced users or AI agents to understand, reproduce, and extend the workflow reliably.
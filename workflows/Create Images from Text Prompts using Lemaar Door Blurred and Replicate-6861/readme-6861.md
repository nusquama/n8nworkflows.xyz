Create Images from Text Prompts using Lemaar Door Blurred and Replicate

https://n8nworkflows.xyz/workflows/create-images-from-text-prompts-using-lemaar-door-blurred-and-replicate-6861


# Create Images from Text Prompts using Lemaar Door Blurred and Replicate

### 1. Workflow Overview

This n8n workflow automates the process of generating images from text prompts using the "lemaar-door-blurrred" model hosted on the Replicate platform. It is designed for creative users and developers who want to generate AI-driven images based on customizable parameters, handling the full request lifecycle including authentication, parameter setup, API invocation, status polling, success/error handling, and logging.

The workflow is logically divided into these key blocks:

- **1.1 Input Reception and Authentication Setup**  
  Starts the workflow manually and sets the Replicate API token necessary for authenticated requests.

- **1.2 Parameter Configuration**  
  Prepares all input parameters for the image generation request, combining required and optional settings.

- **1.3 Image Generation Request and Logging**  
  Sends the generation request to the Replicate API, logs the request for monitoring, and initiates status polling.

- **1.4 Status Polling and Conditional Branching**  
  Periodically checks the status of the generation job, looping with waits as needed, and routes to success or failure handling.

- **1.5 Success and Error Handling**  
  Formats and outputs structured responses based on the generation result.

- **1.6 Result Output**  
  Prepares the final output object for downstream use or display.

- **1.7 Documentation and User Guidance**  
  Includes sticky notes with detailed descriptions, parameter references, usage instructions, and support contacts.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Authentication Setup

- **Overview:**  
  Initiates the workflow manually and sets the required Replicate API token for authentication.

- **Nodes Involved:**  
  - Manual Trigger  
  - Set API Token

- **Node Details:**  

  - **Manual Trigger**  
    - Type: Trigger  
    - Role: Entry point to start the workflow manually.  
    - Config: No parameters, simply a manual start.  
    - Inputs: None  
    - Outputs: Connects to "Set API Token"  
    - Failure cases: None expected.

  - **Set API Token**  
    - Type: Set  
    - Role: Stores the Replicate API token in a workflow variable for later use.  
    - Config: The token is set as a string named "api_token" with a placeholder value "YOUR_REPLICATE_API_TOKEN" which must be replaced by the user.  
    - Inputs: From Manual Trigger  
    - Outputs: To "Set Other Parameters"  
    - Failure cases: Missing or invalid token will cause authentication failure downstream.

---

#### 2.2 Parameter Configuration

- **Overview:**  
  Collects and configures all parameters required by the model for image generation, including prompt, image mask, dimensions, and generation controls.

- **Nodes Involved:**  
  - Set Other Parameters

- **Node Details:**  

  - **Set Other Parameters**  
    - Type: Set  
    - Role: Prepares a comprehensive set of input parameters for the API request, pulling the API token from the previous node to pass along.  
    - Config:  
      - Copies "api_token" from "Set API Token" node.  
      - Sets default values for parameters such as mask (placeholder image URL), seed (-1 for random), input image URL, model version ("dev"), width and height (512), prompt text ("Create something amazing"), and multiple other generation-specific options (e.g., go_fast, lora_scale, num_inference_steps).  
    - Inputs: From "Set API Token"  
    - Outputs: To "Create Other Prediction"  
    - Failure cases: Invalid parameter values can cause API request failure or unexpected behavior.

---

#### 2.3 Image Generation Request and Logging

- **Overview:**  
  Sends the generation request to Replicate API with the configured parameters and logs request metadata for monitoring.

- **Nodes Involved:**  
  - Create Other Prediction  
  - Log Request  
  - Wait 5s

- **Node Details:**  

  - **Create Other Prediction**  
    - Type: HTTP Request  
    - Role: Performs POST request to Replicate API endpoint `/v1/predictions` to start image generation.  
    - Config:  
      - URL: `https://api.replicate.com/v1/predictions`  
      - Method: POST  
      - Headers: Authorization Bearer token set from "api_token", Prefer header set to "wait" for synchronous response.  
      - Body: JSON object containing model version and all input parameters injected from "Set Other Parameters" node.  
      - Response handling: Never error, response parsed as JSON.  
    - Inputs: From "Set Other Parameters"  
    - Outputs: To "Log Request"  
    - Failure cases: Auth errors, invalid parameters, API rate limits, network issues.

  - **Log Request**  
    - Type: Code  
    - Role: Logs request metadata (timestamp, prediction ID, model type) to console for debugging and monitoring.  
    - Config: JavaScript code accessing input JSON, logging to console, then passing data through.  
    - Inputs: From "Create Other Prediction"  
    - Outputs: To "Wait 5s"  
    - Failure cases: None critical; logging failures do not block workflow.

  - **Wait 5s**  
    - Type: Wait  
    - Role: Pauses workflow for 5 seconds before status check to allow processing time.  
    - Inputs: From "Log Request"  
    - Outputs: To "Check Status"  
    - Failure cases: None.

---

#### 2.4 Status Polling and Conditional Branching

- **Overview:**  
  Implements a polling loop that checks the prediction status periodically, waiting and retrying on incomplete or failed states.

- **Nodes Involved:**  
  - Check Status  
  - Is Complete?  
  - Has Failed?  
  - Wait 10s

- **Node Details:**  

  - **Check Status**  
    - Type: HTTP Request  
    - Role: Fetches the current status of the prediction from Replicate API using prediction ID.  
    - Config:  
      - URL dynamically constructed using prediction ID from "Create Other Prediction" node.  
      - Method: GET  
      - Headers: Authorization Bearer token from "Set API Token" node.  
      - Response handling: Never error, JSON response.  
    - Inputs: From "Wait 5s" and "Wait 10s"  
    - Outputs: To "Is Complete?"  
    - Failure cases: Network errors, auth issues.

  - **Is Complete?**  
    - Type: If  
    - Role: Checks if the prediction status equals "succeeded".  
    - Config: Condition: `$json.status == "succeeded"`  
    - Inputs: From "Check Status"  
    - Outputs:  
      - True branch: To "Success Response"  
      - False branch: To "Has Failed?"  
    - Failure cases: Expression evaluation errors if status field missing.

  - **Has Failed?**  
    - Type: If  
    - Role: Checks if the prediction status equals "failed".  
    - Config: Condition: `$json.status == "failed"`  
    - Inputs: From "Is Complete?" (false branch)  
    - Outputs:  
      - True branch: To "Error Response"  
      - False branch: To "Wait 10s" (loop continues)  
    - Failure cases: Expression errors, infinite loops if status stuck unknown.

  - **Wait 10s**  
    - Type: Wait  
    - Role: Waits 10 seconds before retrying the status check.  
    - Inputs: From "Has Failed?" (false branch)  
    - Outputs: To "Check Status"  
    - Failure cases: None.

---

#### 2.5 Success and Error Handling

- **Overview:**  
  Constructs structured JSON responses for both successful and failed generation attempts for downstream processing or user feedback.

- **Nodes Involved:**  
  - Success Response  
  - Error Response

- **Node Details:**  

  - **Success Response**  
    - Type: Set  
    - Role: Creates a JSON object marking success with relevant information (result URL, prediction ID, status, message).  
    - Config: Sets "response" to an object containing success=true, output URL, prediction ID, status, and a success message.  
    - Inputs: From "Is Complete?" (true branch)  
    - Outputs: To "Display Result"  
    - Failure cases: Missing expected fields in input JSON.

  - **Error Response**  
    - Type: Set  
    - Role: Creates a JSON object marking failure with error details and status.  
    - Config: Sets "response" to an object containing success=false, error message (from input or default), prediction ID, status, and failure message.  
    - Inputs: From "Has Failed?" (true branch)  
    - Outputs: To "Display Result"  
    - Failure cases: Missing error info in input may reduce detail.

---

#### 2.6 Result Output

- **Overview:**  
  Prepares the final output object with the response data for consumption by downstream nodes or external systems.

- **Nodes Involved:**  
  - Display Result

- **Node Details:**  

  - **Display Result**  
    - Type: Set  
    - Role: Wraps the "response" object into a "final_result" property for a clean output interface.  
    - Config: Sets "final_result" field with value from incoming "response" JSON.  
    - Inputs: From "Success Response" or "Error Response"  
    - Outputs: None (final output)  
    - Failure cases: None.

---

#### 2.7 Documentation and User Guidance

- **Overview:**  
  Provides extensive sticky notes with introduction, detailed model and parameter descriptions, instructions, troubleshooting, and support contacts.

- **Nodes Involved:**  
  - Sticky Note9  
  - Sticky Note4

- **Node Details:**  

  - **Sticky Note9**  
    - Type: Sticky Note  
    - Role: Displays contact info and social links for support and tutorials.  
    - Content highlights:  
      - Contact email: Yaron@nofluff.online  
      - YouTube channel and LinkedIn profile links.

  - **Sticky Note4**  
    - Type: Sticky Note  
    - Role: Comprehensive documentation including:  
      - Model overview and API endpoint  
      - Parameter reference with required/optional notes  
      - Workflow components explanation  
      - Key benefits  
      - Quick start instructions  
      - Troubleshooting guide  
      - Useful external links:  
        - Model docs: https://replicate.com/creativeathive/lemaar-door-blurrred  
        - Replicate API docs: https://replicate.com/docs  
        - n8n docs: https://docs.n8n.io

---

### 3. Summary Table

| Node Name           | Node Type           | Functional Role                      | Input Node(s)           | Output Node(s)              | Sticky Note                                                                                                                                            |
|---------------------|---------------------|------------------------------------|------------------------|----------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------|
| Manual Trigger      | Manual Trigger       | Starts workflow                    | None                   | Set API Token               |                                                                                                                                                        |
| Set API Token       | Set                  | Stores Replicate API token         | Manual Trigger          | Set Other Parameters        |                                                                                                                                                        |
| Set Other Parameters| Set                  | Configures generation parameters   | Set API Token            | Create Other Prediction     |                                                                                                                                                        |
| Create Other Prediction | HTTP Request       | Sends generation request to API    | Set Other Parameters     | Log Request                 |                                                                                                                                                        |
| Log Request         | Code                 | Logs request info for monitoring   | Create Other Prediction  | Wait 5s                    |                                                                                                                                                        |
| Wait 5s             | Wait                 | Pauses before status check         | Log Request              | Check Status                |                                                                                                                                                        |
| Check Status        | HTTP Request         | Retrieves current prediction status| Wait 5s, Wait 10s        | Is Complete?                |                                                                                                                                                        |
| Is Complete?        | If                   | Checks if generation succeeded     | Check Status             | Success Response, Has Failed? |                                                                                                                                                        |
| Has Failed?         | If                   | Checks if generation failed        | Is Complete?             | Error Response, Wait 10s    |                                                                                                                                                        |
| Wait 10s            | Wait                 | Waits before retrying status check | Has Failed?              | Check Status                |                                                                                                                                                        |
| Success Response    | Set                  | Prepares success response JSON     | Is Complete?             | Display Result              |                                                                                                                                                        |
| Error Response      | Set                  | Prepares error response JSON       | Has Failed?              | Display Result              |                                                                                                                                                        |
| Display Result      | Set                  | Outputs final response object      | Success Response, Error Response | None                 |                                                                                                                                                        |
| Sticky Note9        | Sticky Note          | Support contact and social links   | None                    | None                       | =======================================<br>LEMAAR-DOOR-BLURRRED GENERATOR<br>For any questions or support, please contact:<br>Yaron@nofluff.online<br>Explore more tips and tutorials here:<br>- YouTube: https://www.youtube.com/@YaronBeen/videos<br>- LinkedIn: https://www.linkedin.com/in/yaronbeen/ |
| Sticky Note4        | Sticky Note          | Full documentation and instructions| None                    | None                       | See detailed documentation, parameter reference, quick start, troubleshooting, and useful links at:<br>https://replicate.com/creativeathive/lemaar-door-blurrred |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Position: Start of workflow  
   - No configuration needed.

2. **Create Set API Token Node**  
   - Type: Set  
   - Connect input from Manual Trigger  
   - Add a string field named `api_token` with value `"YOUR_REPLICATE_API_TOKEN"` (replace with actual token before running).  
   - Output connects to next node.

3. **Create Set Other Parameters Node**  
   - Type: Set  
   - Connect input from Set API Token  
   - Add fields with these default values:  
     - `api_token`: Copy from `Set API Token` node (`={{ $('Set API Token').item.json.api_token }}`)  
     - `mask`: `"https://via.placeholder.com/512x512/000000/FFFFFF.png"`  
     - `seed`: `-1` (integer)  
     - `image`: `"https://picsum.photos/512/512"`  
     - `model`: `"dev"`  
     - `width`: `512`  
     - `height`: `512`  
     - `prompt`: `"Create something amazing"`  
     - `go_fast`: `false` (boolean)  
     - `extra_lora`: `""`  
     - `lora_scale`: `1`  
     - `megapixels`: `"1"`  
     - `num_outputs`: `1`  
     - `aspect_ratio`: `"1:1"`  
     - `output_format`: `"webp"`  
     - `guidance_scale`: `3`  
     - `output_quality`: `80`  
     - `prompt_strength`: `0.8`  
     - `extra_lora_scale`: `1`  
     - `replicate_weights`: `""`  
     - `num_inference_steps`: `28`  
     - `disable_safety_checker`: `false`

4. **Create Create Other Prediction Node (HTTP Request)**  
   - Type: HTTP Request  
   - Connect input from Set Other Parameters  
   - Configure:  
     - URL: `https://api.replicate.com/v1/predictions`  
     - Method: POST  
     - Headers:  
       - `Authorization`: `Bearer {{ $json.api_token }}`  
       - `Prefer`: `wait`  
     - Body Content Type: JSON  
     - JSON Body: Use all parameters from the input JSON as the "input" object following the model version `"creativeathive/lemaar-door-blurrred:0e435a9516abafe4df04d7057bccacbb598d674db68316ea2397dc6363f62963"`  
     - Response Format: JSON  
     - Enable "Never Error" to handle possible API errors gracefully.

5. **Create Log Request Node (Code)**  
   - Type: Code  
   - Connect input from Create Other Prediction  
   - Use JavaScript code to log: timestamp, prediction ID, and model type ("other")  
   - Pass through all input data unchanged.

6. **Create Wait 5s Node**  
   - Type: Wait  
   - Connect input from Log Request  
   - Configure to wait 5 seconds.

7. **Create Check Status Node (HTTP Request)**  
   - Type: HTTP Request  
   - Connect input from Wait 5s (also will connect from Wait 10s later)  
   - Configure:  
     - URL: `https://api.replicate.com/v1/predictions/{{ $('Create Other Prediction').item.json.id }}`  
     - Method: GET  
     - Header: `Authorization: Bearer {{ $('Set API Token').item.json.api_token }}`  
     - Response Format: JSON  
     - Never Error enabled.

8. **Create Is Complete? Node (If)**  
   - Type: If  
   - Connect input from Check Status  
   - Condition: Check if `$json.status == "succeeded"`  
   - True output: Connect to Success Response  
   - False output: Connect to Has Failed?

9. **Create Has Failed? Node (If)**  
   - Type: If  
   - Connect input from Is Complete? (false branch)  
   - Condition: Check if `$json.status == "failed"`  
   - True output: Connect to Error Response  
   - False output: Connect to Wait 10s

10. **Create Wait 10s Node**  
    - Type: Wait  
    - Connect input from Has Failed? (false branch)  
    - Configure to wait 10 seconds  
    - Output connects back to Check Status (loop).

11. **Create Success Response Node (Set)**  
    - Type: Set  
    - Connect input from Is Complete? (true branch)  
    - Configure fields:  
      - `response` (object) containing:  
        - `success`: true  
        - `result_url`: `$json.output` (image URL from API response)  
        - `prediction_id`: `$json.id`  
        - `status`: `$json.status`  
        - `message`: `"Other generated successfully"`

12. **Create Error Response Node (Set)**  
    - Type: Set  
    - Connect input from Has Failed? (true branch)  
    - Configure fields:  
      - `response` (object) containing:  
        - `success`: false  
        - `error`: `$json.error` or string `"Other generation failed"` if missing  
        - `prediction_id`: `$json.id`  
        - `status`: `$json.status`  
        - `message`: `"Failed to generate other"`

13. **Create Display Result Node (Set)**  
    - Type: Set  
    - Connect input from Success Response and Error Response  
    - Configure field:  
      - `final_result` set to the incoming `response` object.

14. **Add Sticky Notes for Documentation**  
    - Create two Sticky Note nodes near the start of the workflow:  
      - One with contact info and social links  
      - One with detailed documentation, parameter descriptions, instructions, troubleshooting, and useful links.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | Context or Link                                                                                                 |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| **LEMAAR-DOOR-BLURRRED GENERATOR**<br>For any questions or support, please contact:<br>Yaron@nofluff.online<br>Explore more tips and tutorials here:<br>- YouTube: https://www.youtube.com/@YaronBeen/videos<br>- LinkedIn: https://www.linkedin.com/in/yaronbeen/                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Sticky Note with support contact and social media links                                                       |
| **Model Documentation:** https://replicate.com/creativeathive/lemaar-door-blurrred<br>**Replicate API Docs:** https://replicate.com/docs<br>**n8n Documentation:** https://docs.n8n.io<br><br>**Key Benefits:**<br>- Instant AI-based image generation<br>- Automated full generation lifecycle<br>- Built-in retry and error handling<br>- Easily customizable parameters<br><br>**Common Troubleshooting:**<br>- Verify your Replicate API token is valid and has quota<br>- Check parameter types and values<br>- Monitor generation times and logs<br>- Keep API tokens secure and do not expose publicly                                                                                                                                                                                                                                                                                                                                          | Sticky Note with detailed documentation, usage instructions, and troubleshooting guidance                    |

---

**Disclaimer:**  
The text provided is derived exclusively from an automated n8n workflow. It complies strictly with content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.
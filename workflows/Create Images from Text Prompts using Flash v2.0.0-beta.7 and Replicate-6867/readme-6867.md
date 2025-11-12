Create Images from Text Prompts using Flash v2.0.0-beta.7 and Replicate

https://n8nworkflows.xyz/workflows/create-images-from-text-prompts-using-flash-v2-0-0-beta-7-and-replicate-6867


# Create Images from Text Prompts using Flash v2.0.0-beta.7 and Replicate

### 1. Workflow Overview

This n8n workflow automates the generation of AI-created images from text prompts using the Flash v2.0.0-beta.7 model via the Replicate API. It targets users seeking to transform textual descriptions into images with customizable parameters such as image size, style, and quality. The workflow logically divides into these blocks:

- **1.1 Input & Configuration:** Receives manual trigger input, sets API authentication token, and consolidates all generation parameters.
- **1.2 AI Generation Request:** Submits the image generation request to the Replicate API and receives a prediction ID.
- **1.3 Polling & Status Checking:** Periodically checks the prediction status until completion or failure.
- **1.4 Result Handling:** Processes success or failure cases, formats responses accordingly.
- **1.5 Logging & Output:** Logs generation requests for monitoring and outputs final results.
- **1.6 Documentation & Support:** Provides embedded documentation and support information through sticky notes.

---

### 2. Block-by-Block Analysis

#### 2.1 Input & Configuration

**Overview:**  
This block initiates the workflow manually, sets the Replicate API token, and defines all necessary parameters for the image generation model.

**Nodes Involved:**  
- Manual Trigger  
- Set API Token  
- Set Other Parameters  

**Node Details:**

- **Manual Trigger**  
  - Type: Trigger node (manual)  
  - Role: Starts the workflow execution on user command  
  - Configuration: No parameters; manual start only  
  - Input: None  
  - Output: Triggers next node  
  - Edge Cases: None specific; user must initiate manually  

- **Set API Token**  
  - Type: Set node  
  - Role: Defines the API token required for authentication with Replicate API  
  - Configuration: Stores token in variable `api_token` with placeholder value `"YOUR_REPLICATE_API_TOKEN"` to be replaced by user  
  - Input: Manual Trigger output  
  - Output: Passes token to parameter-setting node  
  - Edge Cases: Failure if token is invalid or missing; user must replace placeholder before execution  

- **Set Other Parameters**  
  - Type: Set node  
  - Role: Aggregates all model input parameters, including the token from the previous node and extensive generation settings like prompt, image URLs, size, quality, and model options  
  - Configuration:  
    - Copies `api_token` dynamically from previous node  
    - Sets default values for 20+ parameters such as:  
      - `mask` (URL for image mask)  
      - `seed` (-1 for random)  
      - `image` (URL for base image)  
      - `model` (model variant, default "dev")  
      - `width` & `height` (512 each)  
      - `prompt` ("Create something amazing")  
      - `go_fast` (false)  
      - Other parameters controlling output format, guidance scale, inference steps, safety checker, etc.  
  - Input: API token from Set API Token node  
  - Output: Prepared JSON body for prediction request  
  - Edge Cases: Parameters must conform to expected types; invalid URLs or unsupported options may cause API errors  

---

#### 2.2 AI Generation Request

**Overview:**  
This block sends the prepared image generation request to Replicate API, initiating the prediction process and returning a prediction ID.

**Nodes Involved:**  
- Create Other Prediction  
- Log Request  

**Node Details:**

- **Create Other Prediction**  
  - Type: HTTP Request node  
  - Role: Sends POST request to Replicate API `/v1/predictions` endpoint to start image generation  
  - Configuration:  
    - URL: `https://api.replicate.com/v1/predictions`  
    - Method: POST  
    - Headers: Authorization header with Bearer token from input JSON `api_token`  
    - Body: JSON constructed from all parameters in `Set Other Parameters` node  
    - Wait preference: `"Prefer": "wait"` used to wait for initial response  
    - Response: JSON, with `neverError` to prevent node failure on HTTP errors  
  - Input: Parameters JSON  
  - Output: Prediction response JSON containing `id` (prediction ID), status, and other metadata  
  - Edge Cases:  
    - API token invalid or expired leads to 401 errors  
    - Bad parameters cause 400 errors  
    - Network issues or API downtime  
  - Requirements: Valid API token and parameter correctness  

- **Log Request**  
  - Type: Code node (JavaScript)  
  - Role: Logs prediction request details for monitoring  
  - Configuration: Logs timestamp, prediction ID, and model type to console  
  - Input: Output of Create Other Prediction node  
  - Output: Passes data unchanged onward  
  - Edge Cases: Logging failures do not affect workflow execution  

---

#### 2.3 Polling & Status Checking

**Overview:**  
This block implements a polling loop that waits and checks the prediction status until it either succeeds or fails.

**Nodes Involved:**  
- Wait 5s  
- Check Status  
- Is Complete?  
- Has Failed?  
- Wait 10s  

**Node Details:**

- **Wait 5s**  
  - Type: Wait node  
  - Role: Pauses execution for 5 seconds before status check  
  - Configuration: 5 seconds delay  
  - Input: Output of Log Request  
  - Output: Triggers Check Status node  
  - Edge Cases: Delays longer than necessary may slow workflow  

- **Check Status**  
  - Type: HTTP Request node  
  - Role: Queries the current status of the prediction using its ID  
  - Configuration:  
    - URL: `https://api.replicate.com/v1/predictions/{{ prediction_id }}` dynamically injected from `Create Other Prediction` node  
    - Method: GET (default)  
    - Headers: Authorization with API token from `Set API Token` node  
    - Response: JSON, never error enabled  
  - Input: After Wait 5s or Wait 10s  
  - Output: Current prediction status JSON  
  - Edge Cases: API downtime, invalid prediction ID, or token errors  

- **Is Complete?**  
  - Type: If node  
  - Role: Checks if prediction status equals `"succeeded"`  
  - Configuration: Condition: `$json.status === 'succeeded'`  
  - Input: Check Status output  
  - Output:  
    - True: Routes to Success Response node  
    - False: Routes to Has Failed? node  

- **Has Failed?**  
  - Type: If node  
  - Role: Checks if prediction status equals `"failed"`  
  - Configuration: Condition: `$json.status === 'failed'`  
  - Input: Is Complete? false branch  
  - Output:  
    - True: Routes to Error Response node  
    - False: Routes to Wait 10s node for retry  

- **Wait 10s**  
  - Type: Wait node  
  - Role: Pauses for 10 seconds before rechecking status to implement retry delay  
  - Configuration: 10 seconds delay  
  - Input: Has Failed? false branch  
  - Output: Loops back to Check Status for polling  
  - Edge Cases: Excessive delays increase total workflow runtime  

---

#### 2.4 Result Handling

**Overview:**  
This block formats and outputs the workflow's final response depending on success or failure.

**Nodes Involved:**  
- Success Response  
- Error Response  
- Display Result  

**Node Details:**

- **Success Response**  
  - Type: Set node  
  - Role: Formats a JSON object indicating success with prediction details and generated image URLs  
  - Configuration: Sets `response` object with keys:  
    - `success: true`  
    - `result_url` (links to generated images)  
    - `prediction_id` and `status`  
    - `message` confirming success  
  - Input: Is Complete? true branch  
  - Output: Passes formatted response to Display Result  

- **Error Response**  
  - Type: Set node  
  - Role: Formats a JSON object indicating failure with error info  
  - Configuration: Sets `response` object with keys:  
    - `success: false`  
    - `error` (from JSON or generic message)  
    - `prediction_id` and `status`  
    - `message` confirming failure  
  - Input: Has Failed? true branch  
  - Output: Passes formatted response to Display Result  

- **Display Result**  
  - Type: Set node  
  - Role: Assigns the final response object to `final_result` for output or further use  
  - Configuration: Copies incoming `response` object to `final_result`  
  - Input: Both Success Response and Error Response outputs converge here  
  - Output: Final structured output of the workflow  

---

#### 2.5 Logging & Output

**Overview:**  
Handles logging of requests and outputs the final structured result object.

**Nodes Involved:**  
- Log Request (described above)  
- Display Result (described above)  

**Node Details:**  
Covered in previous sections.

---

#### 2.6 Documentation & Support

**Overview:**  
Provides embedded documentation, support contact, and usage instructions as sticky notes within the workflow canvas.

**Nodes Involved:**  
- Sticky Note9  
- Sticky Note4  

**Node Details:**

- **Sticky Note9**  
  - Type: Sticky Note  
  - Content: Contact info for support, including email and social media links (YouTube, LinkedIn)  
  - Purpose: User support and community engagement  

- **Sticky Note4**  
  - Type: Sticky Note  
  - Content: Detailed workflow overview, parameter documentation, quick start, troubleshooting, and links to external resources such as Replicate API docs and n8n documentation  
  - Purpose: Self-contained documentation for users and maintainers  

---

### 3. Summary Table

| Node Name              | Node Type            | Functional Role                          | Input Node(s)             | Output Node(s)             | Sticky Note                                                                                                    |
|------------------------|----------------------|----------------------------------------|---------------------------|----------------------------|---------------------------------------------------------------------------------------------------------------|
| Manual Trigger         | manualTrigger        | Initiates the workflow manually        | None                      | Set API Token              |                                                                                                               |
| Set API Token          | set                  | Sets Replicate API token                | Manual Trigger            | Set Other Parameters       |                                                                                                               |
| Set Other Parameters   | set                  | Defines all generation parameters       | Set API Token             | Create Other Prediction    |                                                                                                               |
| Create Other Prediction| httpRequest          | Sends generation request to Replicate   | Set Other Parameters      | Log Request                |                                                                                                               |
| Log Request            | code                 | Logs prediction request details          | Create Other Prediction   | Wait 5s                    |                                                                                                               |
| Wait 5s                | wait                 | Waits 5 seconds before status check     | Log Request               | Check Status               |                                                                                                               |
| Check Status           | httpRequest          | Checks prediction status                 | Wait 5s, Wait 10s         | Is Complete?               |                                                                                                               |
| Is Complete?           | if                   | Checks if prediction succeeded          | Check Status              | Success Response, Has Failed? |                                                                                                               |
| Has Failed?            | if                   | Checks if prediction failed             | Is Complete? (false)      | Error Response, Wait 10s   |                                                                                                               |
| Wait 10s               | wait                 | Waits 10 seconds before retry            | Has Failed? (false)       | Check Status               |                                                                                                               |
| Success Response       | set                  | Formats successful response JSON         | Is Complete? (true)       | Display Result             |                                                                                                               |
| Error Response         | set                  | Formats error response JSON              | Has Failed? (true)        | Display Result             |                                                                                                               |
| Display Result         | set                  | Outputs final response object             | Success Response, Error Response | None                  |                                                                                                               |
| Sticky Note9           | stickyNote            | Provides support contact info             | None                      | None                      | Contact: Yaron@nofluff.online; YouTube & LinkedIn links                                                       |
| Sticky Note4           | stickyNote            | Embedded detailed documentation          | None                      | None                      | Comprehensive model overview, parameters, troubleshooting, resources, and quick start instructions           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Manual Trigger node:**  
   - Set no parameters. This node starts the workflow manually.

3. **Add a Set node named "Set API Token":**  
   - Add a string field named `api_token`.  
   - Set its value to your Replicate API token (replace `"YOUR_REPLICATE_API_TOKEN"`).  
   - Connect Manual Trigger output to this node.

4. **Add another Set node named "Set Other Parameters":**  
   - Add all the following fields with these default values, pulling `api_token` dynamically from "Set API Token":  
     - `api_token`: `={{ $('Set API Token').item.json.api_token }}` (expression)  
     - `mask`: `"https://via.placeholder.com/512x512/000000/FFFFFF.png"`  
     - `seed`: `-1`  
     - `image`: `"https://picsum.photos/512/512"`  
     - `model`: `"dev"`  
     - `width`: `512`  
     - `height`: `512`  
     - `prompt`: `"Create something amazing"`  
     - `go_fast`: `false` (boolean)  
     - `extra_lora`: `""` (empty string)  
     - `lora_scale`: `1`  
     - `megapixels`: `"1"`  
     - `num_outputs`: `1`  
     - `aspect_ratio`: `"1:1"`  
     - `output_format`: `"webp"`  
     - `guidance_scale`: `3`  
     - `output_quality`: `80`  
     - `prompt_strength`: `0.8`  
     - `extra_lora_scale`: `1`  
     - `replicate_weights`: `""` (empty string)  
     - `num_inference_steps`: `28`  
     - `disable_safety_checker`: `false` (boolean)  
   - Connect "Set API Token" output to this node.

5. **Add an HTTP Request node named "Create Other Prediction":**  
   - HTTP Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Headers:  
     - `Authorization`: `Bearer {{$json.api_token}}` (expression)  
     - `Prefer`: `wait`  
   - Body: Raw JSON with the model version and inputs, using expressions to map all parameters from the incoming JSON, for example:  
     ```json
     {
       "version": "settyan/flash-v2.0.0-beta.7:054a842ef89d949dd0dbead5df7607cd43adedc7319aabd56fe6fdfeac8a8b5e",
       "input": {
         "mask": "{{ $json.mask }}",
         "seed": {{ $json.seed }},
         "image": "{{ $json.image }}",
         "model": "{{ $json.model }}",
         "width": {{ $json.width }},
         "height": {{ $json.height }},
         "prompt": "{{ $json.prompt }}",
         "go_fast": {{ $json.go_fast }},
         "extra_lora": "{{ $json.extra_lora }}",
         "lora_scale": {{ $json.lora_scale }},
         "megapixels": "{{ $json.megapixels }}",
         "num_outputs": {{ $json.num_outputs }},
         "aspect_ratio": "{{ $json.aspect_ratio }}",
         "output_format": "{{ $json.output_format }}",
         "guidance_scale": {{ $json.guidance_scale }},
         "output_quality": {{ $json.output_quality }},
         "prompt_strength": {{ $json.prompt_strength }},
         "extra_lora_scale": {{ $json.extra_lora_scale }},
         "replicate_weights": "{{ $json.replicate_weights }}",
         "num_inference_steps": {{ $json.num_inference_steps }},
         "disable_safety_checker": {{ $json.disable_safety_checker }}
       }
     }
     ```  
   - Enable JSON body and never error on response  
   - Connect "Set Other Parameters" output to this node.

6. **Add a Code node named "Log Request":**  
   - JavaScript code to log details:  
     ```js
     const data = $input.all()[0].json;
     console.log('settyan/flash-v2.0.0-beta.7 Request:', {
       timestamp: new Date().toISOString(),
       prediction_id: data.id,
       model_type: 'other'
     });
     return $input.all();
     ```  
   - Connect "Create Other Prediction" output to this node.

7. **Add a Wait node named "Wait 5s":**  
   - Wait for 5 seconds  
   - Connect "Log Request" output to this node.

8. **Add an HTTP Request node named "Check Status":**  
   - Method: GET (default)  
   - URL: `https://api.replicate.com/v1/predictions/{{ $('Create Other Prediction').item.json.id }}` (expression)  
   - Headers: `Authorization` with Bearer token from "Set API Token" node  
   - Enable JSON response, never error  
   - Connect "Wait 5s" and later "Wait 10s" outputs to this node.

9. **Add an If node named "Is Complete?":**  
   - Condition: `$json.status === 'succeeded'`  
   - Connect "Check Status" output to this node.

10. **Add an If node named "Has Failed?":**  
    - Condition: `$json.status === 'failed'`  
    - Connect "Is Complete?" false output to this node.

11. **Add a Set node named "Success Response":**  
    - Set field `response` as object:  
      ```js
      {
        success: true,
        result_url: $json.output,
        prediction_id: $json.id,
        status: $json.status,
        message: 'Other generated successfully'
      }
      ```  
    - Connect "Is Complete?" true output to this node.

12. **Add a Set node named "Error Response":**  
    - Set field `response` as object:  
      ```js
      {
        success: false,
        error: $json.error || 'Other generation failed',
        prediction_id: $json.id,
        status: $json.status,
        message: 'Failed to generate other'
      }
      ```  
    - Connect "Has Failed?" true output to this node.

13. **Add a Wait node named "Wait 10s":**  
    - Wait 10 seconds  
    - Connect "Has Failed?" false output to this node.  
    - Connect "Wait 10s" output back to "Check Status" (loop for polling).

14. **Add a Set node named "Display Result":**  
    - Set field `final_result` to the incoming `response` object.  
    - Connect both "Success Response" and "Error Response" outputs to this node.

15. **Add Sticky Notes (optional):**  
    - Add two sticky notes with detailed documentation and support info as per original content for user assistance.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                         | Context or Link                                                                                     |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| For any questions or support, contact Yaron at Yaron@nofluff.online. Explore tutorials and tips on YouTube: https://www.youtube.com/@YaronBeen/videos and LinkedIn: https://www.linkedin.com/in/yaronbeen/                         | Support contact and community resources                                                          |
| Model Documentation: https://replicate.com/settyan/flash-v2.0.0-beta.7                                                                                                                                                              | Official model page                                                                                |
| Replicate API Documentation: https://replicate.com/docs                                                                                                                                                                            | API reference and usage instructions                                                             |
| n8n Documentation: https://docs.n8n.io                                                                                                                                                                                             | General n8n workflow automation and node instructions                                            |
| Workflow includes comprehensive parameter reference, troubleshooting guide, and quick start instructions embedded as sticky notes for ease of use and maintenance.                                                                | Internal documentation within workflow sticky notes                                               |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow created with n8n, respecting all content policies. It contains no illegal, offensive, or protected elements. All data processed is legal and publicly available.
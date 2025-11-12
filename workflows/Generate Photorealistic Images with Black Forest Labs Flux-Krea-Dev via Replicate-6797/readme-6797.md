Generate Photorealistic Images with Black Forest Labs Flux-Krea-Dev via Replicate

https://n8nworkflows.xyz/workflows/generate-photorealistic-images-with-black-forest-labs-flux-krea-dev-via-replicate-6797


# Generate Photorealistic Images with Black Forest Labs Flux-Krea-Dev via Replicate

### 1. Workflow Overview

This workflow automates the generation of photorealistic images using the Black Forest Labs model "flux-krea-dev" via the Replicate API. It is designed for users who want to convert text prompts and optional image inputs into high-quality AI-generated images seamlessly.

The workflow consists of the following logical blocks:

- **1.1 Input Initialization**: Starts the workflow manually and sets up the API authentication token.
- **1.2 Parameter Configuration**: Defines all parameters for image generation, including prompt, seed, image URL, and output preferences.
- **1.3 Image Generation Request**: Sends a request to the Replicate API to initiate image generation using the configured parameters.
- **1.4 Monitoring & Polling**: Periodically checks the status of the image generation prediction until completion or failure.
- **1.5 Result Handling & Response**: Processes successful or failed generation outcomes, formats response data accordingly.
- **1.6 Logging & Monitoring**: Logs request details for debugging and monitoring purposes.
- **1.7 User Guidance & Documentation**: Contains sticky notes with detailed instructions, parameter explanations, and support contacts.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Initialization

- **Overview:** This block triggers the workflow manually and sets the API token used for authenticating requests to the Replicate API.
- **Nodes Involved:** Manual Trigger, Set API Token
- **Node Details:**

  - **Manual Trigger**
    - Type: Trigger node to start workflow manually.
    - Configuration: No parameters; user clicks to start.
    - Connections: Outputs to "Set API Token".
    - Edge Cases: None; manual action required to start.

  - **Set API Token**
    - Type: Set node for defining static variables.
    - Configuration: Assigns the string `"YOUR_REPLICATE_API_TOKEN"` to the variable `api_token`. This should be replaced by the user’s actual API token.
    - Expressions: None, static assignment.
    - Connections: Receives from "Manual Trigger", outputs to "Set Image Parameters".
    - Edge Cases: Failure if token is invalid or missing when used later; user must replace placeholder token.

#### 2.2 Parameter Configuration

- **Overview:** Prepares all parameters for the image generation request, including required and optional settings with sensible defaults.
- **Nodes Involved:** Set Image Parameters
- **Node Details:**

  - **Set Image Parameters**
    - Type: Set node to assign multiple parameters.
    - Configuration:
      - Copies API token from previous node.
      - Sets defaults such as:
        - `seed`: -1 (random seed)
        - `image`: "https://picsum.photos/512/512" (default input image URL)
        - `prompt`: "A beautiful landscape with mountains and a lake at sunset"
        - `go_fast`: true (speed optimized)
        - `guidance`: 3
        - `megapixels`: "1"
        - `num_outputs`: 1
        - `aspect_ratio`: "1:1"
        - `output_format`: "webp"
        - `output_quality`: 80
        - `prompt_strength`: 0.8
        - `num_inference_steps`: 28
        - `disable_safety_checker`: false
    - Expressions: Copies `api_token` from "Set API Token".
    - Connections: Receives from "Set API Token", outputs to "Create Image Prediction".
    - Edge Cases: Incorrect parameter types or invalid values can cause API errors; user can modify parameters as needed.

#### 2.3 Image Generation Request

- **Overview:** Sends a POST request to Replicate API to create an image generation prediction with the specified parameters.
- **Nodes Involved:** Create Image Prediction
- **Node Details:**

  - **Create Image Prediction**
    - Type: HTTP Request node.
    - Configuration:
      - URL: `https://api.replicate.com/v1/predictions`
      - Method: POST
      - Headers:
        - Authorization: Bearer token from `api_token`
        - Prefer: wait (to wait for immediate prediction if possible)
      - Body: JSON, includes model version and all parameters from the input.
      - Response Format: JSON, never error to handle failures gracefully.
    - Expressions: Dynamically injects all image parameters from previous node.
    - Connections: Receives from "Set Image Parameters", outputs to "Log Request".
    - Edge Cases:
      - API authentication failure (invalid token).
      - Network errors or timeouts.
      - Parameter validation errors from API.
      - API rate limits.

#### 2.4 Monitoring & Polling

- **Overview:** Implements a loop to wait and poll the prediction status until it is either successful or failed, with retries and wait periods to avoid excessive API calls.
- **Nodes Involved:** Log Request, Wait 5s, Check Status, Is Complete?, Has Failed?, Wait 10s
- **Node Details:**

  - **Log Request**
    - Type: Code node.
    - Configuration: Logs timestamp, prediction ID, and model type to console for monitoring.
    - Input: Receives prediction response.
    - Output: Passes to "Wait 5s".
    - Edge Cases: None critical; logs may fail if console unavailable.

  - **Wait 5s**
    - Type: Wait node.
    - Configuration: Pauses workflow for 5 seconds before next status check.
    - Input: From "Log Request".
    - Output: To "Check Status".
    - Edge Cases: None critical.

  - **Check Status**
    - Type: HTTP Request node.
    - Configuration:
      - URL: `https://api.replicate.com/v1/predictions/{{prediction_id}}` (dynamic).
      - Method: GET (default).
      - Headers: Authorization with API token.
      - Response: JSON, never error.
    - Input: From "Wait 5s" and "Wait 10s".
    - Output: To "Is Complete?".
    - Edge Cases:
      - Network failure.
      - API errors.
      - Invalid prediction ID.

  - **Is Complete?**
    - Type: If node.
    - Configuration: Checks if `"status"` equals `"succeeded"`.
    - Input: From "Check Status".
    - Output:
      - True: To "Success Response".
      - False: To "Has Failed?".
    - Edge Cases: Unexpected status values.

  - **Has Failed?**
    - Type: If node.
    - Configuration: Checks if `"status"` equals `"failed"`.
    - Input: From "Is Complete?" false branch.
    - Output:
      - True: To "Error Response".
      - False: To "Wait 10s" (retry status check).
    - Edge Cases: Infinite loop if status is stuck in unknown state.

  - **Wait 10s**
    - Type: Wait node.
    - Configuration: Pauses 10 seconds before retrying status check.
    - Input: From "Has Failed?" false branch.
    - Output: Back to "Check Status".
    - Edge Cases: None critical; longer wait to reduce API load.

#### 2.5 Result Handling & Response

- **Overview:** Formats the final response object based on success or error, then prepares output data for downstream consumption or user retrieval.
- **Nodes Involved:** Success Response, Error Response, Display Result
- **Node Details:**

  - **Success Response**
    - Type: Set node.
    - Configuration: Creates an object with keys `success: true`, `image_url`, `prediction_id`, `status`, and a success message.
    - Input: From "Is Complete?" true branch.
    - Output: To "Display Result".
    - Edge Cases: Missing output URL in response.

  - **Error Response**
    - Type: Set node.
    - Configuration: Creates an object with keys `success: false`, `error` message, `prediction_id`, `status`, and a failure message.
    - Input: From "Has Failed?" true branch.
    - Output: To "Display Result".
    - Edge Cases: Missing error details in response.

  - **Display Result**
    - Type: Set node.
    - Configuration: Assigns the response object (success or error) to the variable `final_result`.
    - Input: From both "Success Response" and "Error Response".
    - Output: End node, delivers final workflow output.
    - Edge Cases: None critical.

#### 2.6 Logging & Monitoring

- **Overview:** Provides a central logging point for requests for audit and debugging.
- **Nodes Involved:** Log Request
- **Node Details:** See details in 2.4.

#### 2.7 User Guidance & Documentation

- **Overview:** Contains two large sticky notes with extensive documentation, instructions, parameter explanations, troubleshooting, and contact information.
- **Nodes Involved:** Sticky Note4, Sticky Note9
- **Node Details:**

  - **Sticky Note4**
    - Purpose: Full model overview, parameter reference, workflow explanation, benefits, quick start, troubleshooting, and useful links.
    - Location: Positioned prominently for user visibility.
    - Content highlights:
      - Model owner, description, and API endpoint.
      - Detailed parameter explanations with defaults.
      - Workflow component descriptions.
      - Quick start steps to configure and run.
      - Troubleshooting tips and best practices.
      - Links to official Replicate and n8n documentation.

  - **Sticky Note9**
    - Purpose: Contact and support information for the workflow author.
    - Content includes:
      - Contact email: Yaron@nofluff.online
      - YouTube and LinkedIn links for tutorials and support.

---

### 3. Summary Table

| Node Name              | Node Type          | Functional Role                            | Input Node(s)                 | Output Node(s)                 | Sticky Note                                                                                   |
|------------------------|--------------------|--------------------------------------------|------------------------------|-------------------------------|-----------------------------------------------------------------------------------------------|
| Manual Trigger         | Manual Trigger      | Initiates workflow execution               |                              | Set API Token                 |                                                                                               |
| Set API Token          | Set                | Stores Replicate API token                  | Manual Trigger               | Set Image Parameters          |                                                                                               |
| Set Image Parameters   | Set                | Configures image generation parameters     | Set API Token                | Create Image Prediction       |                                                                                               |
| Create Image Prediction| HTTP Request       | Sends image generation request to API      | Set Image Parameters         | Log Request                  |                                                                                               |
| Log Request            | Code               | Logs request details for monitoring         | Create Image Prediction      | Wait 5s                      |                                                                                               |
| Wait 5s                | Wait               | Pauses 5 seconds before status check        | Log Request                 | Check Status                 |                                                                                               |
| Check Status           | HTTP Request       | Queries prediction status                    | Wait 5s, Wait 10s           | Is Complete?                 |                                                                                               |
| Is Complete?           | If                 | Checks if prediction succeeded               | Check Status                | Success Response, Has Failed?|                                                                                               |
| Has Failed?            | If                 | Checks if prediction failed                  | Is Complete?                | Error Response, Wait 10s     |                                                                                               |
| Wait 10s               | Wait               | Pauses 10 seconds before retrying status    | Has Failed?                 | Check Status                 |                                                                                               |
| Success Response       | Set                | Formats success output response              | Is Complete? (True)          | Display Result               |                                                                                               |
| Error Response         | Set                | Formats error output response                | Has Failed? (True)           | Display Result               |                                                                                               |
| Display Result         | Set                | Prepares final output object                  | Success Response, Error Response |                           |                                                                                               |
| Sticky Note4           | Sticky Note        | Detailed workflow documentation and resources|                              |                               | Contains extensive instructions, parameter guide, and troubleshooting info                    |
| Sticky Note9           | Sticky Note        | Author contact information and support links |                              |                               | Contact: Yaron@nofluff.online, YouTube & LinkedIn links                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**
   - Type: Manual Trigger
   - Purpose: Start the workflow manually.

2. **Create Set Node "Set API Token"**
   - Type: Set
   - Add string field `api_token`.
   - Value: `"YOUR_REPLICATE_API_TOKEN"` (replace with actual token).
   - Connect output of Manual Trigger to this node.

3. **Create Set Node "Set Image Parameters"**
   - Type: Set
   - Add multiple fields:
     - `api_token` (string): `={{ $('Set API Token').item.json.api_token }}`
     - `seed` (number): `-1`
     - `image` (string): `"https://picsum.photos/512/512"`
     - `prompt` (string): `"A beautiful landscape with mountains and a lake at sunset"`
     - `go_fast` (boolean): `true`
     - `guidance` (number): `3`
     - `megapixels` (string): `"1"`
     - `num_outputs` (number): `1`
     - `aspect_ratio` (string): `"1:1"`
     - `output_format` (string): `"webp"`
     - `output_quality` (number): `80`
     - `prompt_strength` (number): `0.8`
     - `num_inference_steps` (number): `28`
     - `disable_safety_checker` (boolean): `false`
   - Connect output of "Set API Token" to this node.

4. **Create HTTP Request Node "Create Image Prediction"**
   - Type: HTTP Request
   - Method: POST
   - URL: `https://api.replicate.com/v1/predictions`
   - Headers:
     - `Authorization`: `Bearer {{$json.api_token}}`
     - `Prefer`: `wait`
   - Body Type: JSON
   - Body Content:
     ```json
     {
       "version": "black-forest-labs/flux-krea-dev:ce472e62d34a1f4e5415eb704a032ecf118f067345ef4a9cc1913d01e369b7a3",
       "input": {
         "seed": {{$json.seed}},
         "image": "{{$json.image}}",
         "prompt": "{{$json.prompt}}",
         "go_fast": {{$json.go_fast}},
         "guidance": {{$json.guidance}},
         "megapixels": "{{$json.megapixels}}",
         "num_outputs": {{$json.num_outputs}},
         "aspect_ratio": "{{$json.aspect_ratio}}",
         "output_format": "{{$json.output_format}}",
         "output_quality": {{$json.output_quality}},
         "prompt_strength": {{$json.prompt_strength}},
         "num_inference_steps": {{$json.num_inference_steps}},
         "disable_safety_checker": {{$json.disable_safety_checker}}
       }
     }
     ```
   - Response Format: JSON, Never Error enabled.
   - Connect output of "Set Image Parameters" to this node.

5. **Create Code Node "Log Request"**
   - Type: Code
   - JavaScript code:
     ```javascript
     const data = $input.all()[0].json;
     console.log('black-forest-labs/flux-krea-dev Request:', {
       timestamp: new Date().toISOString(),
       prediction_id: data.id,
       model_type: 'image'
     });
     return $input.all();
     ```
   - Connect output of "Create Image Prediction" to this node.

6. **Create Wait Node "Wait 5s"**
   - Type: Wait
   - Duration: 5 seconds.
   - Connect output of "Log Request" to this node.

7. **Create HTTP Request Node "Check Status"**
   - Type: HTTP Request
   - Method: GET (default)
   - URL: `https://api.replicate.com/v1/predictions/{{ $('Create Image Prediction').item.json.id }}`
   - Headers:
     - `Authorization`: `Bearer {{ $('Set API Token').item.json.api_token }}`
   - Response Format: JSON, Never Error enabled.
   - Connect output of "Wait 5s" and also from "Wait 10s" (created later) to this node.

8. **Create If Node "Is Complete?"**
   - Type: If
   - Condition: Check if `$json.status === 'succeeded'`
   - Connect output of "Check Status" to this node.

9. **Create If Node "Has Failed?"**
   - Type: If
   - Condition: Check if `$json.status === 'failed'`
   - Connect false branch of "Is Complete?" to this node.

10. **Create Set Node "Success Response"**
    - Type: Set
    - Assign `response` object:
      ```json
      {
        "success": true,
        "image_url": $json.output,
        "prediction_id": $json.id,
        "status": $json.status,
        "message": "Image generated successfully"
      }
      ```
    - Connect true branch of "Is Complete?" to this node.

11. **Create Set Node "Error Response"**
    - Type: Set
    - Assign `response` object:
      ```json
      {
        "success": false,
        "error": $json.error || "Image generation failed",
        "prediction_id": $json.id,
        "status": $json.status,
        "message": "Failed to generate image"
      }
      ```
    - Connect true branch of "Has Failed?" to this node.

12. **Create Wait Node "Wait 10s"**
    - Type: Wait
    - Duration: 10 seconds.
    - Connect false branch of "Has Failed?" to this node.
    - Connect output back to "Check Status" to continue polling.

13. **Create Set Node "Display Result"**
    - Type: Set
    - Assign `final_result` = `$json.response`
    - Connect both "Success Response" and "Error Response" outputs to this node.

14. **Add Sticky Notes for Documentation**
    - Sticky Note4: Paste the detailed documentation content about model overview, parameters, workflow explanation, troubleshooting, and resources.
    - Sticky Note9: Paste contact and support information, including email and social media links.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Context or Link                                                                                           |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| Workflow powered by Black Forest Labs’ flux-krea-dev model via Replicate API for photorealistic AI image generation. Includes retry logic and structured error handling to ensure robust generation pipeline.                                                                                                                                                                                                                                                                                                                   | Workflow description                                                                                       |
| Contact for support: Yaron@nofluff.online. Explore video tutorials and tips on YouTube: https://www.youtube.com/@YaronBeen/videos and LinkedIn: https://www.linkedin.com/in/yaronbeen/                                                                                                                                                                                                                                                                                                                                      | Sticky Note9                                                                                              |
| Model documentation and API references: https://replicate.com/black-forest-labs/flux-krea-dev and https://replicate.com/docs. n8n official docs at https://docs.n8n.io                                                                                                                                                                                                                                                                                                                                                         | Sticky Note4                                                                                              |
| Common troubleshooting includes verifying API token validity, parameter correctness, monitoring generation time, and respecting API rate limits.                                                                                                                                                                                                                                                                                                                                                                             | Sticky Note4                                                                                              |

---

This documentation fully describes the workflow structure, nodes, logic, and usage, enabling precise understanding, reproduction, and modification for any advanced user or AI agent.
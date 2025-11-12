Generate Videos from Images using Wan 2.2 I2V Fast AI and Replicate API

https://n8nworkflows.xyz/workflows/generate-videos-from-images-using-wan-2-2-i2v-fast-ai-and-replicate-api-6802


# Generate Videos from Images using Wan 2.2 I2V Fast AI and Replicate API

---

### 1. Workflow Overview

This workflow automates the generation of videos from single images using the "wan-2.2-i2v-fast" AI model hosted on the Replicate API platform. It enables users to input an image and a descriptive prompt, configure various video generation parameters, send a request to the Replicate API, then poll for completion status, handle success or failure responses gracefully, and finally output the generated video URL or error details.

The workflow is designed for scenarios where fast, cost-efficient AI-powered video synthesis from images is required, such as creative content generation, prototyping, or automation pipelines involving media assets.

**Logical Blocks:**

- **1.1 Input Initialization:** Starting point and setting up API credentials and video generation parameters.
- **1.2 Video Generation Request:** Submitting the video generation job to Replicate API.
- **1.3 Status Polling Loop:** Waiting and repeatedly checking the job status until completion or failure.
- **1.4 Result Handling:** Processing successful outputs or errors, formatting response data.
- **1.5 Logging & Output:** Logging request data and preparing final workflow output.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Input Initialization

**Overview:**  
This block initiates the workflow manually, sets the Replicate API token for authentication, and configures all input parameters required by the AI model for video generation.

**Nodes Involved:**  
- Manual Trigger  
- Set API Token  
- Set Video Parameters  

**Node Details:**  

- **Manual Trigger**  
  - Type: Trigger  
  - Role: Starts the workflow manually on user command.  
  - Configuration: Default manual trigger, no parameters.  
  - Inputs: None  
  - Outputs: Triggers "Set API Token" node.  
  - Edge cases: None expected; user must manually trigger workflow.

- **Set API Token**  
  - Type: Set  
  - Role: Stores the Replicate API token securely for reuse.  
  - Configuration: Assigns a string value `"YOUR_REPLICATE_API_TOKEN"` to variable `api_token`. This should be replaced with a valid token.  
  - Inputs: From Manual Trigger  
  - Outputs: Passes `api_token` to "Set Video Parameters" node.  
  - Edge cases: Invalid or missing token will cause authentication errors downstream.

- **Set Video Parameters**  
  - Type: Set  
  - Role: Defines all parameters for video generation, including prompt, image URL, and model options.  
  - Configuration:
    - `api_token`: References token from "Set API Token" node.
    - `seed`: -1 (random seed)
    - `image`: Default image URL `"https://picsum.photos/512/512"`
    - `prompt`: `"A person walking through a magical forest with glowing particles"`
    - `go_fast`: true (enables faster generation)
    - `num_frames`: 81 (number of frames to generate)
    - `resolution`: `"480p"`
    - `aspect_ratio`: `"16:9"`
    - `sample_shift`: 12
    - `frames_per_second`: 16  
  - Inputs: From "Set API Token"  
  - Outputs: Passes parameters to "Create Video Prediction" node.  
  - Edge cases: Invalid parameter types or values may cause API rejection or unexpected results.

---

#### 1.2 Video Generation Request

**Overview:**  
This block sends the configured video generation request to the Replicate API and initiates the prediction job.

**Nodes Involved:**  
- Create Video Prediction  
- Log Request  

**Node Details:**  

- **Create Video Prediction**  
  - Type: HTTP Request  
  - Role: Sends POST request to Replicate API to start video generation.  
  - Configuration:
    - URL: `https://api.replicate.com/v1/predictions`
    - Method: POST
    - Headers: Authorization Bearer token from `api_token` variable; Prefer header set to "wait" for synchronous-like behavior.
    - Body (JSON): Includes version id of the model and input parameters as defined in "Set Video Parameters".
    - Response: JSON, with error suppression (`neverError: true`) to handle failures gracefully.
  - Inputs: From "Set Video Parameters"  
  - Outputs: Passes prediction response to "Log Request" node.  
  - Edge cases:  
    - Authentication failure if token invalid.  
    - API rate limits or quota exceeded errors.  
    - Network timeouts or unexpected response formats.

- **Log Request**  
  - Type: Code (JavaScript)  
  - Role: Logs prediction request details (timestamp, prediction ID, model type) for monitoring/debugging.  
  - Configuration: Uses console.log with structured data.  
  - Inputs: From "Create Video Prediction"  
  - Outputs: Triggers "Wait 5s" node for status polling.  
  - Edge cases: Logging errors unlikely but if console unavailable, logs may be lost.

---

#### 1.3 Status Polling Loop

**Overview:**  
This block implements a loop to wait and check the prediction status repeatedly until the video generation completes or fails.

**Nodes Involved:**  
- Wait 5s  
- Check Status  
- Is Complete?  
- Has Failed?  
- Wait 10s  

**Node Details:**  

- **Wait 5s**  
  - Type: Wait  
  - Role: Pauses the workflow for 5 seconds before checking status.  
  - Configuration: Wait time set to 5 seconds.  
  - Inputs: From "Log Request" or "Wait 10s" (retry).  
  - Outputs: Triggers "Check Status".  
  - Edge cases: Minimal risk, but excessive polling could cause delays or API throttling.

- **Check Status**  
  - Type: HTTP Request  
  - Role: Queries Replicate API for the current status of the prediction job.  
  - Configuration:
    - URL: Dynamic, uses prediction ID from "Create Video Prediction".
    - Method: GET
    - Headers: Authorization Bearer token from "Set API Token".
    - Response: JSON, error suppression enabled.  
  - Inputs: From "Wait 5s" or "Wait 10s"  
  - Outputs: Passes status response to "Is Complete?"  
  - Edge cases:  
    - Network errors.  
    - Invalid prediction ID.  
    - API rate limiting.

- **Is Complete?**  
  - Type: If  
  - Role: Checks if the prediction status is `"succeeded"`.  
  - Configuration: Condition checks if `$json.status == "succeeded"`.  
  - Inputs: From "Check Status"  
  - Outputs:  
    - True branch: "Success Response" node  
    - False branch: "Has Failed?" node  
  - Edge cases: Status field missing or unexpected value.

- **Has Failed?**  
  - Type: If  
  - Role: Checks if the prediction status is `"failed"`.  
  - Configuration: Condition checks if `$json.status == "failed"`.  
  - Inputs: From "Is Complete?" (False branch)  
  - Outputs:  
    - True branch: "Error Response" node  
    - False branch: "Wait 10s" node to retry status check  
  - Edge cases: API returning unknown statuses or transient errors.

- **Wait 10s**  
  - Type: Wait  
  - Role: Pauses the workflow for 10 seconds before retrying status check.  
  - Configuration: Wait time set to 10 seconds.  
  - Inputs: From "Has Failed?" (False branch)  
  - Outputs: Triggers "Check Status" to continue polling.  
  - Edge cases: Excessive retries could delay workflow termination.

---

#### 1.4 Result Handling

**Overview:**  
Processes the final prediction output or error and structures a uniform JSON response.

**Nodes Involved:**  
- Success Response  
- Error Response  
- Display Result  

**Node Details:**  

- **Success Response**  
  - Type: Set  
  - Role: Constructs a JSON object signaling success with video URL and metadata.  
  - Configuration:  
    - Sets `response` field with an object containing:  
      - `success: true`  
      - `video_url`: URL from prediction output  
      - `prediction_id`, `status`, and a success message  
  - Inputs: From "Is Complete?" (True branch)  
  - Outputs: Passes to "Display Result".  
  - Edge cases: Missing output URL or malformed data.

- **Error Response**  
  - Type: Set  
  - Role: Constructs a JSON object signaling failure with error details.  
  - Configuration:  
    - Sets `response` field with an object containing:  
      - `success: false`  
      - `error`: error message or fallback string  
      - `prediction_id`, `status`, and failure message  
  - Inputs: From "Has Failed?" (True branch)  
  - Outputs: Passes to "Display Result".  
  - Edge cases: Missing error info or unexpected structure.

- **Display Result**  
  - Type: Set  
  - Role: Copies the final response object into a standardized field `final_result` for output.  
  - Configuration: Directly assigns `final_result` from previous node's `response`.  
  - Inputs: From both "Success Response" and "Error Response"  
  - Outputs: Final node of the workflow, provides output for downstream usage or UI display.  
  - Edge cases: None significant.

---

#### 1.5 Logging & Documentation

**Overview:**  
Additional nodes provide metadata, user instructions, and contact information as sticky notes embedded within the workflow for user guidance.

**Nodes Involved:**  
- Sticky Note9  
- Sticky Note4  

**Node Details:**  

- **Sticky Note9**  
  - Type: Sticky Note  
  - Role: Displays contact info for support and links to video tutorials and LinkedIn profile of the author.  
  - Content Highlights:  
    - Contact: Yaron@nofluff.online  
    - YouTube and LinkedIn URLs for support and tutorials.  
  - Position: Near the workflow start for visibility.

- **Sticky Note4**  
  - Type: Sticky Note  
  - Role: Provides comprehensive workflow overview, parameter guide, troubleshooting, and step-by-step instructions.  
  - Content Highlights:  
    - Model details and API documentation links  
    - Parameter descriptions and defaults  
    - Workflow step explanations  
    - Troubleshooting tips and best practices  
    - Quick start instructions  
  - Purpose: Acts as embedded documentation for users maintaining or modifying the workflow.

---

### 3. Summary Table

| Node Name             | Node Type          | Functional Role                                  | Input Node(s)                     | Output Node(s)                    | Sticky Note                                                                                                  |
|-----------------------|--------------------|-------------------------------------------------|----------------------------------|----------------------------------|--------------------------------------------------------------------------------------------------------------|
| Manual Trigger        | Manual Trigger     | Starts workflow execution manually               | None                             | Set API Token                    |                                                                                                              |
| Set API Token         | Set                | Stores Replicate API token                        | Manual Trigger                  | Set Video Parameters             |                                                                                                              |
| Set Video Parameters  | Set                | Sets all video generation input parameters       | Set API Token                   | Create Video Prediction          |                                                                                                              |
| Create Video Prediction| HTTP Request       | Sends video generation request to Replicate API  | Set Video Parameters            | Log Request                     |                                                                                                              |
| Log Request           | Code (JavaScript)  | Logs prediction request details                   | Create Video Prediction         | Wait 5s                        |                                                                                                              |
| Wait 5s               | Wait               | Pauses for 5 seconds before status check          | Log Request, Wait 10s           | Check Status                   |                                                                                                              |
| Check Status          | HTTP Request       | Retrieves prediction status from Replicate API    | Wait 5s                        | Is Complete?                   |                                                                                                              |
| Is Complete?          | If                 | Checks if prediction succeeded                     | Check Status                   | Success Response, Has Failed?   |                                                                                                              |
| Has Failed?           | If                 | Checks if prediction failed                         | Is Complete?                   | Error Response, Wait 10s        |                                                                                                              |
| Wait 10s              | Wait               | Pauses 10 seconds before retrying status check    | Has Failed?                    | Check Status                   |                                                                                                              |
| Success Response      | Set                | Formats success JSON response                       | Is Complete? (True branch)     | Display Result                 |                                                                                                              |
| Error Response        | Set                | Formats error JSON response                         | Has Failed? (True branch)       | Display Result                 |                                                                                                              |
| Display Result        | Set                | Outputs final structured result                     | Success Response, Error Response| None                         |                                                                                                              |
| Sticky Note9          | Sticky Note        | Contact/support info and social media links       | None                          | None                          | Contact and support info: Yaron@nofluff.online; YouTube and LinkedIn links provided.                         |
| Sticky Note4          | Sticky Note        | Detailed workflow documentation and parameter guide| None                          | None                          | Comprehensive workflow overview, parameter details, quick start, troubleshooting, and links to docs.          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Manual Trigger node:**  
   - Name: `Manual Trigger`  
   - Purpose: To start the workflow manually.  
   - Leave default settings.

3. **Add a Set node for API Token:**  
   - Name: `Set API Token`  
   - Add a string field `api_token`.  
   - Set value to `YOUR_REPLICATE_API_TOKEN` (replace with actual token).  
   - Connect output of `Manual Trigger` to this node.

4. **Add a Set node for Video Parameters:**  
   - Name: `Set Video Parameters`  
   - Add fields:  
     - `api_token`: Expression referencing `Set API Token` node’s `api_token` (e.g., `={{ $('Set API Token').item.json.api_token }}`)  
     - `seed`: Number, default `-1` (random seed)  
     - `image`: String, default `"https://picsum.photos/512/512"`  
     - `prompt`: String, default `"A person walking through a magical forest with glowing particles"`  
     - `go_fast`: Boolean, default `true`  
     - `num_frames`: Number, default `81`  
     - `resolution`: String, default `"480p"`  
     - `aspect_ratio`: String, default `"16:9"`  
     - `sample_shift`: Number, default `12`  
     - `frames_per_second`: Number, default `16`  
   - Connect output of `Set API Token` to this node.

5. **Add HTTP Request node to create prediction:**  
   - Name: `Create Video Prediction`  
   - Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Headers:  
     - Authorization: `Bearer {{ $json.api_token }}`  
     - Prefer: `wait`  
   - Body (JSON):  
     ```json
     {
       "version": "wan-video/wan-2.2-i2v-fast:309fbbedb17f8e9c50f6141a13c27e2ce71cf11bf78f01419f05f334ece056b5",
       "input": {
         "seed": {{ $json.seed }},
         "image": "{{ $json.image }}",
         "prompt": "{{ $json.prompt }}",
         "go_fast": {{ $json.go_fast }},
         "num_frames": {{ $json.num_frames }},
         "resolution": "{{ $json.resolution }}",
         "aspect_ratio": "{{ $json.aspect_ratio }}",
         "sample_shift": {{ $json.sample_shift }},
         "frames_per_second": {{ $json.frames_per_second }}
       }
     }
     ```  
   - Connect output of `Set Video Parameters` to this node.

6. **Add a Code node to log request info:**  
   - Name: `Log Request`  
   - Language: JavaScript  
   - Code:
     ```javascript
     const data = $input.all()[0].json;

     console.log('wan-video/wan-2.2-i2v-fast Request:', {
       timestamp: new Date().toISOString(),
       prediction_id: data.id,
       model_type: 'video'
     });

     return $input.all();
     ```  
   - Connect output of `Create Video Prediction` to this node.

7. **Add a Wait node:**  
   - Name: `Wait 5s`  
   - Wait time: 5 seconds  
   - Connect output of `Log Request` to this node.

8. **Add HTTP Request node to check prediction status:**  
   - Name: `Check Status`  
   - Method: GET  
   - URL: `https://api.replicate.com/v1/predictions/{{ $('Create Video Prediction').item.json.id }}`  
   - Header: Authorization Bearer token from `Set API Token` node.  
   - Connect output of `Wait 5s` to this node (also will connect retry later).

9. **Add If node to check for success:**  
   - Name: `Is Complete?`  
   - Condition: `$json.status` equals `"succeeded"`  
   - Connect output of `Check Status` to this node.

10. **Add If node to check for failure:**  
    - Name: `Has Failed?`  
    - Condition: `$json.status` equals `"failed"`  
    - Connect the False branch of `Is Complete?` to this node.

11. **Add Wait node for retry delay:**  
    - Name: `Wait 10s`  
    - Wait time: 10 seconds  
    - Connect False branch of `Has Failed?` to this node.  
    - Connect output of `Wait 10s` back to `Check Status` node to loop polling.

12. **Add Set node for success response:**  
    - Name: `Success Response`  
    - Set field `response` with object:  
      ```javascript
      {
        success: true,
        video_url: $json.output,
        prediction_id: $json.id,
        status: $json.status,
        message: 'Video generated successfully'
      }
      ```  
    - Connect True branch of `Is Complete?` to this node.

13. **Add Set node for error response:**  
    - Name: `Error Response`  
    - Set field `response` with object:  
      ```javascript
      {
        success: false,
        error: $json.error || 'Video generation failed',
        prediction_id: $json.id,
        status: $json.status,
        message: 'Failed to generate video'
      }
      ```  
    - Connect True branch of `Has Failed?` to this node.

14. **Add Set node to display final result:**  
    - Name: `Display Result`  
    - Set field `final_result` to the JSON from previous response node (`Success Response` or `Error Response`).  
    - Connect outputs of both `Success Response` and `Error Response` to this node.

15. **Add Sticky Notes** (optional but recommended):  
    - Add notes with contact info, parameter guide, instructions, and links to model/API docs. Use the content from Sticky Note4 and Sticky Note9 nodes.

16. **Save and Test:**  
    - Replace API token placeholder with a valid Replicate API token.  
    - Adjust parameters as needed and trigger manually.  
    - Monitor outputs and logs for successful video URL or error messages.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                            | Context or Link                                                    |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------|
| Workflow powered by Replicate API and n8n automation, designed for fast video generation from images using the wan-2.2-i2v-fast AI model.                                                                                                                                 | General workflow description                                     |
| Contact for support: Yaron@nofluff.online                                                                                                                                                                                                                              | Support email                                                    |
| Video tutorials and tips available on YouTube: https://www.youtube.com/@YaronBeen/videos                                                                                                                                                                                | Video tutorials                                                  |
| Professional profile and updates: https://www.linkedin.com/in/yaronbeen/                                                                                                                                                                                                | LinkedIn profile                                                |
| Official model documentation: https://replicate.com/wan-video/wan-2.2-i2v-fast                                                                                                                                                                                          | Model docs                                                      |
| Replicate API documentation: https://replicate.com/docs                                                                                                                                                                                                                | API docs                                                        |
| n8n official documentation: https://docs.n8n.io                                                                                                                                                                                                                        | n8n platform docs                                               |
| Troubleshooting tips: Validate API token, check parameters, monitor for generation timeouts, and verify output format. Use default parameters initially for best stability. Keep API tokens secure.                                                                        | Troubleshooting instructions                                    |

---

**Disclaimer:**  
The text provided is extracted exclusively from an automated workflow created with n8n, a powerful integration and automation tool. The workflow strictly respects relevant content policies and contains no illegal, offensive, or protected content. All data processed is legal and publicly accessible.
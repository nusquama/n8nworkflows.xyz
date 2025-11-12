Schedule & Publish All Instagram Content Types with Facebook Graph API

https://n8nworkflows.xyz/workflows/schedule---publish-all-instagram-content-types-with-facebook-graph-api-4498


# Schedule & Publish All Instagram Content Types with Facebook Graph API

### 1. Workflow Overview

This workflow automates the scheduling and publishing of all Instagram content types using the Facebook Graph API. It is designed to handle various post formats, including images, stories (image and video), reels, and carousels, by dynamically routing content to the appropriate API endpoints. The workflow is structured around key logical blocks:

- **1.1 Input Setup and Routing:** Manual trigger initiates the workflow; post settings are configured with content metadata and post type; a smart router directs content to the correct publishing pipeline based on post type.
  
- **1.2 Container Creation and Processing:** For each content type, a media container is created through either HTTP requests or Facebook Graph API nodes; the workflow waits and polls until the container processing is complete and ready for publishing.
  
- **1.3 Publishing:** Once the media container is ready, the workflow routes the publishing request either via HTTP API or Facebook SDK nodes based on the content source.
  
- **1.4 Carousel Post Handling:** Special multi-step handling for carousel posts involves creating multiple image containers, then creating the carousel container referencing those images.
  
- **1.5 Wait and Retry Logic:** Implemented to handle asynchronous media processing by waiting initially and retrying status checks until the container reports a “FINISHED” state.

The workflow integrates authentication management using HTTP Header Auth credentials and Facebook Graph API OAuth2 credentials to securely interact with Instagram via Facebook’s platform.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Setup and Routing

**Overview:**  
This block initializes the workflow and configures the input parameters that define the Instagram post’s content and type. It uses a switch node to route the content dynamically to the appropriate processing pipeline based on the post type prefix.

**Nodes Involved:**  
- 🚀 Manual Trigger - Start Workflow  
- ⚙️ Configure Post Settings  
- 🔀 Smart Content Router  

**Node Details:**

- **🚀 Manual Trigger - Start Workflow**  
  - Type: Manual Trigger  
  - Role: Entry point to start the workflow manually.  
  - Configuration: No parameters; triggers manually.  
  - Inputs: None  
  - Outputs: Connects to “Configure Post Settings”.  
  - Edge Cases: None (manual trigger).  

- **⚙️ Configure Post Settings**  
  - Type: Set  
  - Role: Defines key variables such as `node` (Instagram user or page ID), `post_type` (content type), media URLs (`image_url`, `video_url`, `cover_image`), and `caption`.  
  - Configuration: Static assignment of values; can be adapted for dynamic input.  
  - Key Expressions: Direct assignment of properties used downstream.  
  - Inputs: From manual trigger.  
  - Outputs: Connects to “Smart Content Router”.  
  - Edge Cases: Incorrect or missing values here will propagate errors downstream.  

- **🔀 Smart Content Router**  
  - Type: Switch  
  - Role: Routes the workflow flow based on the `post_type` field prefix (e.g., `http_image`, `fb_story_video`) to respective container creation nodes.  
  - Configuration: Multiple conditions matching `post_type` values, each renamed for clarity (e.g., "HTTP - Image", "FB - Reels").  
  - Inputs: From post settings.  
  - Outputs: Multiple branches for different content types and APIs.  
  - Edge Cases: Unrecognized `post_type` causes no route and workflow halt or error.  

---

#### 2.2 Container Creation and Processing

**Overview:**  
Creates a media container on Instagram through either HTTP requests or Facebook Graph API calls, depending on the route selected. After creation, it initiates a wait and polling cycle to check for processing completion.

**Nodes Involved:**  
- Container HTTP Image  
- Container FB Image  
- Container HTTP Story Image  
- Container FB Story Image  
- Container HTTP Story Video  
- Container FB Story Video  
- Container HTTP Reels  
- Container FB Reels  
- Container Carousel Image 1  
- Container Carousel Image 2  
- Container HTTP Carousel  
- 🔍 Check Processing Status  
- 🔍 Check Processing Status1  
- ⏰ Initial Processing Wait  
- ⏰ Retry Wait Loop  
- 🔍 Check Processing Status (set node for container_id)  
- ✅ Is Container Ready?  

**Node Details:**

- **Container * Nodes (HTTP and FB variants)**  
  - Type: HTTP Request or Facebook Graph API  
  - Role: Create media containers by sending POST requests with appropriate parameters (image_url, video_url, caption, media_type, cover_url).  
  - Configuration:  
    - URL templated with `node` (Instagram user/page ID).  
    - Query parameters set dynamically from input JSON (e.g., `image_url`, `caption`).  
    - Authentication: HTTP Header Auth for HTTP requests (custom token), Facebook OAuth2 for Graph API nodes.  
  - Inputs: From “Smart Content Router”.  
  - Outputs: Connect to “🔍 Check Processing Status” (or subsequent container image nodes in carousel case).  
  - Edge Cases:  
    - Authentication failures (credential invalidation).  
    - API rate limits or version deprecations (v22.0 used).  
    - Invalid URLs or media types causing API errors.  
    - Network timeouts or malformed requests.  

- **Container Carousel Image 1 & 2, Container HTTP Carousel**  
  - Special handling for carousel posts: create individual image containers first, then create a carousel container referencing children IDs.  
  - Configuration: Carousel container includes `children` parameter listing container IDs of the images.  
  - Inputs/Outputs chained sequentially to build the carousel.  
  - Edge Cases: Failure to create any child container blocks carousel creation; IDs must be correctly referenced using expressions.  

- **🔍 Check Processing Status & 🔍 Check Processing Status1**  
  - Type: HTTP Request / Set  
  - Role: Poll Instagram/Facebook API for container processing status using container ID.  
  - Configuration: Requests field `status_code` to check if processing is “FINISHED”.  
  - Inputs: Container creation response with container ID.  
  - Outputs: “⏰ Initial Processing Wait” or “✅ Is Container Ready?” node.  
  - Edge Cases:  
    - API errors or container not found.  
    - Status never reaches “FINISHED” (infinite loop risk mitigated by wait and retry).  

- **⏰ Initial Processing Wait & ⏰ Retry Wait Loop**  
  - Type: Wait  
  - Role: Pauses workflow to allow asynchronous container processing.  
  - Configuration: Default wait duration (not explicitly set; manual webhook based).  
  - Inputs: From status check nodes.  
  - Outputs: Loop back to status check nodes for retry.  
  - Edge Cases: Long processing times may cause delays; no explicit timeout or max retries configured.  

- **✅ Is Container Ready?**  
  - Type: If  
  - Role: Checks if container status is “FINISHED” to proceed with publishing or retry waiting.  
  - Inputs: From “🔍 Check Processing Status1”.  
  - Outputs: If true, to “🔀 HTTP vs FB API Router”; if false, to “⏰ Retry Wait Loop”.  
  - Edge Cases: Case sensitivity or missing status_code can cause incorrect routing.  

---

#### 2.3 Publishing

**Overview:**  
Publishes the processed media container as an Instagram post by calling the media_publish endpoint. It selects the publishing method (HTTP API or Facebook SDK) based on the earlier routing decision.

**Nodes Involved:**  
- 🔀 HTTP vs FB API Router  
- 📤 Publish via HTTP API  
- 📤 Publish via Facebook SDK  

**Node Details:**

- **🔀 HTTP vs FB API Router**  
  - Type: If  
  - Role: Determines whether to publish via HTTP API or Facebook SDK based on `post_type` prefix.  
  - Configuration: Checks if prefix is “http” for HTTP API path, else Facebook SDK.  
  - Inputs: From “✅ Is Container Ready?”.  
  - Outputs: Two branches leading to respective publishing nodes.  
  - Edge Cases: Unknown prefix leads to no publishing action.  

- **📤 Publish via HTTP API**  
  - Type: HTTP Request  
  - Role: Publishes the media container by sending POST to `/media_publish` endpoint with `creation_id`.  
  - Configuration: URL and query parameter `creation_id` from container response.  
  - Authentication: HTTP Header Auth.  
  - Inputs: From HTTP branch of router.  
  - Outputs: End of workflow for this path.  
  - Edge Cases: API errors or invalid creation_id cause publish failure.  

- **📤 Publish via Facebook SDK**  
  - Type: Facebook Graph API  
  - Role: Similar publishing via Facebook SDK node using OAuth2 credentials.  
  - Configuration: Uses `media_publish` edge with `creation_id`.  
  - Inputs: From Facebook SDK branch of router.  
  - Outputs: End of workflow for this path.  
  - Edge Cases: OAuth token expiry or API errors.  

---

#### 2.4 Carousel Post Handling (Specific)

**Overview:**  
Handles Instagram carousel posts by creating each image container individually, then creating a parent carousel container referencing all child media IDs.

**Nodes Involved:**  
- Container Carousel Image 1  
- Container Carousel Image 2  
- Container HTTP Carousel  

**Node Details:**  

- **Container Carousel Image 1 & 2**  
  - Type: HTTP Request  
  - Role: Creates individual image containers for carousel children.  
  - Configuration: Uses `image_url` and `caption` from current or routed JSON data.  
  - Inputs: From “Smart Content Router” (HTTP - Carousel branch).  
  - Outputs: Sequentially chained to next image container and then carousel container.  
  - Edge Cases: Failure to create any child container blocks carousel creation.  

- **Container HTTP Carousel**  
  - Type: HTTP Request  
  - Role: Creates the carousel container referencing child container IDs via `children` query parameter.  
  - Configuration: `children` builds a comma-separated list of container IDs using expressions from previous nodes.  
  - Inputs: From “Container Carousel Image 2”.  
  - Outputs: To “🔍 Check Processing Status” for polling.  
  - Edge Cases: Incorrect ID referencing or API errors block carousel publishing.  

---

#### 2.5 General Workflow Controls and Metadata

**Sticky Notes:**  
- Provide user-friendly labels and explanations for the blocks:
  - Image Post Creation  
  - Story Image Upload  
  - Story Video Upload  
  - Instagram Reels Upload  
  - Carousel Post (Multiple Images)  
  - Variable and Routing Setup  
  - Container Processing Pipeline (Create, Wait, Check, Retry)  
  - Smart Publishing System (HTTP or Facebook SDK routing)  
  - Creator Information with social links and documentation references.

---

### 3. Summary Table

| Node Name                    | Node Type               | Functional Role                         | Input Node(s)                | Output Node(s)                   | Sticky Note                                                                                                             |
|------------------------------|-------------------------|--------------------------------------|-----------------------------|---------------------------------|-------------------------------------------------------------------------------------------------------------------------|
| 🚀 Manual Trigger - Start Workflow | Manual Trigger          | Workflow entry point                  | None                        | ⚙️ Configure Post Settings      |                                                                                                                         |
| ⚙️ Configure Post Settings    | Set                     | Defines input variables for post      | 🚀 Manual Trigger            | 🔀 Smart Content Router          |                                                                                                                         |
| 🔀 Smart Content Router       | Switch                  | Routes workflow based on post_type    | ⚙️ Configure Post Settings    | Various container creation nodes | ## Variable and Routing Setup                                                                                           |
| Container HTTP Image          | HTTP Request            | Creates image media container (HTTP) | 🔀 Smart Content Router      | 🔍 Check Processing Status       | ## 📸 Image Post Creation                                                                                                |
| Container FB Image            | Facebook Graph API      | Creates image media container (FB)   | 🔀 Smart Content Router      | 🔍 Check Processing Status       | ## 📸 Image Post Creation                                                                                                |
| Container HTTP Story Image    | HTTP Request            | Creates story image container (HTTP) | 🔀 Smart Content Router      | 🔍 Check Processing Status       | ## 📱 Story Image Upload                                                                                                |
| Container FB Story Image      | Facebook Graph API      | Creates story image container (FB)   | 🔀 Smart Content Router      | 🔍 Check Processing Status       | ## 📱 Story Image Upload                                                                                                |
| Container HTTP Story Video    | HTTP Request            | Creates story video container (HTTP) | 🔀 Smart Content Router      | 🔍 Check Processing Status       | ## 🎬 Story Video Upload                                                                                                |
| Container FB Story Video      | Facebook Graph API      | Creates story video container (FB)   | 🔀 Smart Content Router      | 🔍 Check Processing Status       | ## 🎬 Story Video Upload                                                                                                |
| Container HTTP Reels          | HTTP Request            | Creates reels container (HTTP)        | 🔀 Smart Content Router      | 🔍 Check Processing Status       | ## 🎵 Instagram Reels Upload                                                                                            |
| Container FB Reels            | Facebook Graph API      | Creates reels container (FB)          | 🔀 Smart Content Router      | 🔍 Check Processing Status       | ## 🎵 Instagram Reels Upload                                                                                            |
| Container Carousel Image 1    | HTTP Request            | Creates first image for carousel      | 🔀 Smart Content Router      | Container Carousel Image 2       | ## 🖼️ Carousel Post (Multiple Images)                                                                                   |
| Container Carousel Image 2    | HTTP Request            | Creates second image for carousel     | Container Carousel Image 1   | Container HTTP Carousel          | ## 🖼️ Carousel Post (Multiple Images)                                                                                   |
| Container HTTP Carousel       | HTTP Request            | Creates carousel container            | Container Carousel Image 2   | 🔍 Check Processing Status       | ## 🖼️ Carousel Post (Multiple Images)                                                                                   |
| 🔍 Check Processing Status    | HTTP Request / Set      | Polls container processing status     | Various container nodes      | ⏰ Initial Processing Wait       | ## 🔄 Container Processing Pipeline                                                                                     |
| ⏰ Initial Processing Wait    | Wait                    | Waits before status retry             | 🔍 Check Processing Status   | 🔍 Check Processing Status1      | ## 🔄 Container Processing Pipeline                                                                                     |
| 🔍 Check Processing Status1   | HTTP Request            | Polls status for readiness            | ⏰ Initial Processing Wait    | ✅ Is Container Ready?           | ## 🔄 Container Processing Pipeline                                                                                     |
| ✅ Is Container Ready?        | If                      | Checks if container is ready          | 🔍 Check Processing Status1  | 🔀 HTTP vs FB API Router / ⏰ Retry Wait Loop | ## 🔄 Container Processing Pipeline                                                                                     |
| ⏰ Retry Wait Loop            | Wait                    | Waits before retrying status check   | ✅ Is Container Ready? (False)| 🔍 Check Processing Status1      | ## 🔄 Container Processing Pipeline                                                                                     |
| 🔀 HTTP vs FB API Router      | If                      | Routes to HTTP or FB API publish      | ✅ Is Container Ready? (True)| 📤 Publish via HTTP API / 📤 Publish via Facebook SDK | ## 📤 Smart Publishing System                                                                                           |
| 📤 Publish via HTTP API       | HTTP Request            | Publishes content via HTTP API        | 🔀 HTTP vs FB API Router     | End                            | ## 📤 Smart Publishing System                                                                                           |
| 📤 Publish via Facebook SDK   | Facebook Graph API      | Publishes content via Facebook SDK    | 🔀 HTTP vs FB API Router     | End                            | ## 📤 Smart Publishing System                                                                                           |
| 🔍 Check Processing Status (Set Node) | Set                     | Extracts container_id for status check| Container * nodes           | 🔍 Check Processing Status1      | ## 🔄 Container Processing Pipeline                                                                                     |
| Sticky Notes (various)        | Sticky Note             | Labels and documents workflow blocks | N/A                         | N/A                            | See section 2.5 for content details                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: “🚀 Manual Trigger - Start Workflow”  
   - Type: Manual Trigger (no parameters).  

2. **Create Set Node for Post Settings**  
   - Name: “⚙️ Configure Post Settings”  
   - Type: Set  
   - Assign variables:  
     - `node` (Instagram user/page ID, e.g., “17841456545908024”)  
     - `post_type` (e.g., “http_reel”)  
     - `image_url`, `caption`, `video_url`, `cover_image` (set URLs and text).  
   - Connect manual trigger to this node.  

3. **Create Switch Node for Content Routing**  
   - Name: “🔀 Smart Content Router”  
   - Type: Switch  
   - Create rules matching `post_type` exactly with outputs renamed for clarity:  
     - “http_image”, “fb_image”, “http_story_image”, “fb_story_image”, “http_story_video”, “fb_story_video”, “http_reel”, “fb_reel”, “http_carousel”, “fb_carousel”.  
   - Connect “Configure Post Settings” output to this switch node.  

4. **Create Container Creation Nodes**  
   - For each content type and API (HTTP or FB), create nodes:  
     - HTTP Request Nodes for “Container HTTP Image”, “Container HTTP Story Image”, “Container HTTP Story Video”, “Container HTTP Reels”, “Container HTTP Carousel”, “Container Carousel Image 1”, “Container Carousel Image 2”.  
       - Configure URL as `https://graph.facebook.com/v22.0/{{ $json.node }}/media`  
       - Method: POST  
       - Authentication: HTTP Header Auth (create credential for “lakshitukani IG Account”)  
       - Query parameters set with expressions for media URLs, captions, media_type, cover_url as applicable.  
     - Facebook Graph API Nodes for “Container FB Image”, “Container FB Story Image”, “Container FB Story Video”, “Container FB Reels”.  
       - Edge: media  
       - Node: `={{ $json.node }}`  
       - Method: POST  
       - Authentication: Facebook Graph API OAuth2 credential (create credential “IG lakshitukani Login (n8n Integration 2)”)  
       - Query parameters with relevant media data.  
   - Connect each output of “Smart Content Router” to its respective container creation node(s).  
   - For carousel images: chain “Container Carousel Image 1” → “Container Carousel Image 2” → “Container HTTP Carousel”.  

5. **Create Status Check Logic**  
   - Create a Set node “🔍 Check Processing Status” to store `container_id` from container creation response.  
   - Create HTTP Request nodes “🔍 Check Processing Status” and “🔍 Check Processing Status1” to request container status from `https://graph.facebook.com/v22.0/{{container_id}}?fields=status_code`  
     - Authentication for “Check Processing Status”: HTTP Query Auth credential (create “IG lakshitukani (n8n integration)”).  
   - Create Wait nodes “⏰ Initial Processing Wait” and “⏰ Retry Wait Loop” with reasonable wait times (default or custom).  
   - Create If node “✅ Is Container Ready?” to check if `status_code` equals “FINISHED”.  
   - Connect nodes to implement:  
     - Container creation → Check Processing Status → Initial Wait → Check Processing Status1 → If Is Ready  
     - If ready → HTTP vs FB API Router  
     - If not ready → Retry Wait Loop → Check Processing Status1 (loop).  

6. **Create Publishing Nodes**  
   - Create If node “🔀 HTTP vs FB API Router” checking if prefix of `post_type` is “http” → HTTP API publish else Facebook SDK publish.  
   - Create HTTP Request node “📤 Publish via HTTP API”: POST to `https://graph.facebook.com/v22.0/{{node}}/media_publish` with `creation_id`.  
   - Create Facebook Graph API node “📤 Publish via Facebook SDK”: edge “media_publish”, node `={{ $json.node }}`, POST with `creation_id`.  
   - Connect If node outputs to respective publishing nodes.  

7. **Add Sticky Notes**  
   - Add sticky notes with descriptive titles for each logical block as per Section 2.5.  
   - Include creator info and useful links.  

8. **Credential Setup**  
   - Create HTTP Header Auth credential named “lakshitukani IG Account” with required token.  
   - Create Facebook Graph API OAuth2 credential named “IG lakshitukani Login (n8n Integration 2)” with app credentials and authorization.  
   - Create HTTP Query Auth credential named “IG lakshitukani (n8n integration)” for status polling.  

9. **Test and Debug**  
   - Trigger workflow manually with sample data.  
   - Verify API calls succeed, containers reach “FINISHED” status, and posts are published on Instagram.  
   - Adjust wait times and error handling as needed.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                                                                                                  |
|-------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| Created by Lakshit Ukani. LinkedIn: https://www.linkedin.com/in/lakshit-ukani/                                                       | Creator information and contact.                                                                                               |
| YouTube Channel: https://www.youtube.com/@lakshit-ukani?sub_confirmation=1                                                          | Video tutorials and updates.                                                                                                   |
| Join Community: https://www.skool.com/ai-automation-club-7843                                                                        | Community support for workflow issues and questions.                                                                           |
| Documentation: https://developers.facebook.com/docs/instagram-platform/instagram-graph-api/reference/ig-user/media#creating           | Official Facebook Instagram Graph API reference for media creation and publishing.                                              |
| Workflow uses Facebook Graph API version v22.0 and requires valid Facebook OAuth2 credentials and Instagram Business account IDs.   | API versioning and authentication requirements.                                                                                |
| The workflow handles asynchronous media processing by polling status and waiting with retries to ensure media is ready before publish. | Important for reliability and avoiding premature publish calls.                                                                |

---

This comprehensive reference enables developers and AI agents to understand, reproduce, and extend the Instagram content publishing workflow effectively, anticipating integration and processing challenges inherent to the Facebook Graph API ecosystem.
Schedule Instagram posts at optimal times with Rubinyun

https://n8nworkflows.xyz/workflows/schedule-instagram-posts-at-optimal-times-with-rubinyun-17589


# Schedule Instagram posts at optimal times with Rubinyun

### 1. Workflow Overview

This workflow automates the scheduling and publishing of Instagram posts (images, carousels, reels, or stories) at optimal times using the Rubinyun API. Because the official Instagram API requires immediate publishing and lacks native "schedule for later" capabilities, this workflow offloads queue management and delayed execution to Rubinyun. 

The logical execution is organized into the following functional blocks:
- **1.1 Workflow Initiation:** Manually triggers the workflow execution on demand.
- **1.2 Optimal Time Calculation:** Queries the Rubinyun API to determine the best upcoming posting slot based on the account's historical engagement data while avoiding conflicting scheduled slots.
- **1.3 Post Queueing:** Sends post payloads (including media URL, caption, post type, and calculated timestamp) to the Rubinyun scheduling queue.
- **1.4 Optional Media Upload (Inactive):** An optional, currently disabled utility block designed to upload local binary files to Rubinyun to generate public hosting URLs for media that lacks an external web address.

---

### 2. Block-by-Block Analysis

#### 2.1 Workflow Initiation
- **Overview:** Acts as the manual entry point for the automation sequence.
- **Nodes Involved:** `Run manually to test`
- **Node Details:**
  - **Node Name:** `Run manually to test`
  - **Type and Technical Role:** `n8n-nodes-base.manualTrigger` (v1) — Initiates workflow execution via user interaction in the n8n editor.
  - **Configuration Choices:** Default configurations (no parameters required).
  - **Key Expressions or Variables:** None.
  - **Input/Output Connections:** Output connects to `Get the best time to post`.
  - **Version Requirements:** Version 1.
  - **Edge Cases / Potential Failures:** None (manual execution only).
  - **Sub-workflow Reference:** None.

---

#### 2.2 Optimal Time Calculation
- **Overview:** Connects to the Rubinyun API to evaluate historical engagement patterns and calculate the next best time slot for posting.
- **Nodes Involved:** `Get the best time to post`
- **Node Details:**
  - **Node Name:** `Get the best time to post`
  - **Type and Technical Role:** `n8n-nodes-base.httpRequest` (v4.2) — Performs an HTTP GET request to fetch optimization analytics.
  - **Configuration Choices:** 
    - URL: `https://www.chrononyte.com/rubinyun/api.php`
    - Method: `GET`
    - Authentication: Generic Credential Type (`httpHeaderAuth`)
    - Query Parameters: `action=besttime`, `when=tomorrow`
  - **Key Expressions or Variables:** None in request; outputs JSON containing slot details (`$json.slot.at`).
  - **Input/Output Connections:** Input from `Run manually to test`; output connects to `Schedule the Instagram post`.
  - **Version Requirements:** Version 4.2.
  - **Edge Cases / Potential Failures:** Authentication failures due to incorrect API credentials; network timeouts reaching chrononyte.com.
  - **Sub-workflow Reference:** None.

---

#### 2.3 Post Queueing
- **Overview:** Submits the final post configuration and optimized publishing timestamp to the Rubinyun scheduling queue.
- **Nodes Involved:** `Schedule the Instagram post`
- **Node Details:**
  - **Node Name:** `Schedule the Instagram post`
  - **Type and Technical Role:** `n8n-nodes-base.httpRequest` (v4.2) — Performs an HTTP POST request with form-urlencoded data to add a post to the queue.
  - **Configuration Choices:** 
    - URL: `https://www.chrononyte.com/rubinyun/api.php`
    - Method: `POST`
    - Content Type: `form-urlencoded`
    - Authentication: Generic Credential Type (`httpHeaderAuth`)
    - Query Parameters: `action=add`
    - Body Parameters: 
      - `type`: `image`
      - `image_url`: `https://www.chrononyte.com/assets/demo/rubinyun-demo.jpg`
      - `caption`: `Posted from n8n with Rubinyun`
      - `scheduled_at`: `={{ $json.slot.at }}`
      - `channel`: (Empty string for default profile)
  - **Key Expressions or Variables:** `{{ $json.slot.at }}` (pulls the optimal timestamp from the previous node).
  - **Input/Output Connections:** Input from `Get the best time to post`. Terminal node of the active flow.
  - **Version Requirements:** Version 4.2.
  - **Edge Cases / Potential Failures:** Invalid media URLs, unhandled missing parameters, or invalid time formats if the previous node output fails.
  - **Sub-workflow Reference:** None.

---

#### 2.4 Optional Media Upload (Inactive)
- **Overview:** Provides an optional utility path for uploading local binary files to Rubinyun to generate a temporary public URL when direct external URLs are unavailable.
- **Nodes Involved:** `Optional: upload a media file`
- **Node Details:**
  - **Node Name:** `Optional: upload a media file`
  - **Type and Technical Role:** `n8n-nodes-base.httpRequest` (v4.2) — Performs an HTTP multipart POST request to upload binary files.
  - **Configuration Choices:** 
    - Disabled by default.
    - URL: `https://www.chrononyte.com/rubinyun/upload.php`
    - Method: `POST`
    - Content Type: `multipart-form-data`
    - Authentication: Generic Credential Type (`httpHeaderAuth`)
    - Body Parameters: `image` configured via Form Binary Data (`inputDataFieldName`: `data`)
  - **Key Expressions or Variables:** When active, outputs `{{ $json.url }}` to supply media URLs to downstream scheduling nodes.
  - **Input/Output Connections:** Disabled (not wired into the active execution path).
  - **Version Requirements:** Version 4.2.
  - **Edge Cases / Potential Failures:** Parking limits exceeded (storage space constraints per account tier), unsupported file types, or file size limits exceeded (8 MB for images, 100 MB for videos).
  - **Sub-workflow Reference:** None.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Read me first | `n8n-nodes-base.stickyNote` | Documentation and setup instructions | None | None | ## Schedule and pubish Instagram posts at the best time<br><br>**Who's it for**<br>...<br>(Covers workflow overview and setup steps) |
| Run manually to test | `n8n-nodes-base.manualTrigger` | Workflow execution trigger | None | Get the best time to post | ## Schedule and pubish Instagram posts at the best time<br><br>**Who's it for**<br>... |
| Get the best time to post | `n8n-nodes-base.httpRequest` | Fetch optimal posting time slot | Run manually to test | Schedule the Instagram post | ## Schedule and pubish Instagram posts at the best time<br><br>**Who's it for**<br>... |
| Schedule the Instagram post | `n8n-nodes-base.httpRequest` | Send post payload to Rubinyun queue | Get the best time to post | None | ## Schedule and pubish Instagram posts at the best time<br><br>**Who's it for**<br>... |
| Optional: upload a media file | `n8n-nodes-base.httpRequest` | Upload binary media for public URL | None | None | ### Optional: you have a FILE, not a URL<br><br>Instagram downloads the media from a public address, so a file on your disk cannot<br>be published as it is. Feed this node your file as binary data and enable it: it<br>answers with a public URL to put in the node on the right.<br><br>The file is deleted the moment the post is published, and expires on its own if it<br>never is: it is parking, not hosting. |
| About the optional upload | `n8n-nodes-base.stickyNote` | Upload feature documentation | None | None | ### Optional: you have a FILE, not a URL<br><br>Instagram downloads the media from a public address, so a file on your disk cannot<br>be published as it is. Feed this node your file as binary data and enable it: it<br>answers with a public URL to put in the node on the right.<br><br>The file is deleted the moment the post is published, and expires on its own if it<br>never is: it is parking, not hosting. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node:**
   - Add a **Manual Trigger** node (`n8n-nodes-base.manualTrigger`). Name it `Run manually to test`.
2. **Create the Best Time HTTP Request Node:**
   - Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`). Name it `Get the best time to post`.
   - Set **Method** to `GET`.
   - Set **URL** to `https://www.chrononyte.com/rubinyun/api.php`.
   - Enable **Send Query Parameters**, and add parameters:
     - Name: `action`, Value: `besttime`
     - Name: `when`, Value: `tomorrow`
   - Configure **Authentication** using **Generic Credential Type** -> **Header Auth**.
3. **Create the Post Scheduling HTTP Request Node:**
   - Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`). Name it `Schedule the Instagram post`.
   - Set **Method** to `POST`.
   - Set **URL** to `https://www.chrononyte.com/rubinyun/api.php`.
   - Set **Content Type** to `form-urlencoded`.
   - Enable **Send Query Parameters** with parameter:
     - Name: `action`, Value: `add`
   - Enable **Send Body Parameters**, and add parameters:
     - Name: `type`, Value: `image`
     - Name: `image_url`, Value: `https://www.chrononyte.com/assets/demo/rubinyun-demo.jpg`
     - Name: `caption`, Value: `Posted from n8n with Rubinyun`
     - Name: `scheduled_at`, Value: `={{ $json.slot.at }}`
     - Name: `channel`, Value: (Leave empty)
   - Configure **Authentication** using the same **Header Auth** credential.
4. **Configure Credentials:**
   - Create a new **Header Auth** credential in n8n.
   - Set **Name** to `Authorization`.
   - Set **Value** to `Key YOUR_KEY:YOUR_SECRET` (obtained from your account at `chrononyte.com/rubinyun`).
   - Select this credential in both HTTP Request nodes.
5. **Establish Node Connections:**
   - Connect the output of `Run manually to test` to the input of `Get the best time to post`.
   - Connect the output of `Get the best time to post` to the input of `Schedule the Instagram post`.
6. *(Optional)* **Configure Upload Node (If handling local files):**
   - Add an **HTTP Request** node named `Optional: upload a media file`.
   - Set **Method** to `POST`, **URL** to `https://www.chrononyte.com/rubinyun/upload.php`, and **Content Type** to `multipart-form-data`.
   - Add a body parameter named `image`, set **Parameter Type** to `Form Binary Data`, and set **Input Data Field Name** to `data`.
   - Configure the same **Header Auth** credential and enable the node if required. Update `image_url` in the scheduling node to `{{ $json.url }}`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Rubinyun Account Registration & Setup | [Chrononyte Rubinyun Service](https://www.chrononyte.com/rubinyun) |
| Platform Requirements | Requires an Instagram Business or Creator account linked to a Facebook Page. |
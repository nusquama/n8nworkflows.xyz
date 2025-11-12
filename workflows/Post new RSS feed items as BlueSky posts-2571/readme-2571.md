Post new RSS feed items as BlueSky posts

https://n8nworkflows.xyz/workflows/post-new-rss-feed-items-as-bluesky-posts-2571


# Post new RSS feed items as BlueSky posts

### 1. Workflow Overview

This n8n workflow automates the posting of new RSS feed items as BlueSky posts. It is designed for BlueSky users who want to share content from an RSS feed on their BlueSky timeline without manual effort.

**Target Use Case:**  
Automatically publish new RSS feed entries as posts on BlueSky, including post text (feed content snippet), a link to the original article, and an image either extracted from the feed or a custom one.

**Logical Blocks:**

- **1.1 RSS Feed Input:** Monitors an RSS feed for new items.
- **1.2 BlueSky Authentication:** Creates a session for BlueSky API access.
- **1.3 Timestamp Generation:** Retrieves the current datetime for the post timestamp.
- **1.4 Image Handling:** Downloads the image from the RSS feed enclosure URL and uploads it to BlueSky.
- **1.5 Post Creation:** Constructs and sends the API request to create a new BlueSky post with feed content and embedded image.

---

### 2. Block-by-Block Analysis

#### 2.1 RSS Feed Input

- **Overview:**  
Monitors the specified RSS feed URL for new items on a per-minute polling basis, triggering the workflow when a new item is detected.

- **Nodes Involved:**  
  - RSS Feed Trigger

- **Node Details:**  
  - **RSS Feed Trigger**  
    - **Type:** RSS Feed Read Trigger  
    - **Role:** Entry trigger node that polls the RSS feed URL every minute for new items.  
    - **Configuration:** User must specify the RSS feed URL (`feedUrl` parameter). Polling is set to trigger every minute.  
    - **Key expressions:** Feed URL is static; output contains item fields like `content:encodedSnippet`, `link`, `lintitlek` (likely a typo for `title`), `contentSnippet`, and `enclosure` for image URL/type.  
    - **Connections:** Output triggers "Create Session".  
    - **Edge cases:** If feed URL is invalid, no items will trigger. Feed format inconsistencies or missing fields (e.g., no enclosure) may cause downstream failures.  
    - **Version:** 1

---

#### 2.2 BlueSky Authentication

- **Overview:**  
Creates an authenticated session with BlueSky API using user credentials to obtain an access token for subsequent API calls.

- **Nodes Involved:**  
  - Create Session

- **Node Details:**  
  - **Create Session**  
    - **Type:** HTTP Request  
    - **Role:** Authenticates user with BlueSky by POSTing username and app password to create a session and receive an access JWT and DID (decentralized identifier).  
    - **Configuration:** User must input their BlueSky username and app password (generated via BlueSky app passwords) in the body parameters `identifier` and `password`.  
    - **Key expressions:** None dynamic inside node, but outputs `accessJwt` and `did` used downstream.  
    - **Inputs:** Receives trigger from RSS Feed Trigger.  
    - **Outputs:** Passes session info to "Get current datetime".  
    - **Edge cases:** Authentication failure due to incorrect credentials or network issues; API rate limits.  
    - **Version:** 4.1

---

#### 2.3 Timestamp Generation

- **Overview:**  
Generates the current datetime, used as the post creation timestamp.

- **Nodes Involved:**  
  - Get current datetime

- **Node Details:**  
  - **Get current datetime**  
    - **Type:** DateTime  
    - **Role:** Provides the current date/time in ISO string format for the post's `createdAt` field.  
    - **Configuration:** Default settings, no user input required.  
    - **Inputs:** Receives session node output.  
    - **Outputs:** Passes current date/time to "Download image".  
    - **Edge cases:** Minimal risk; system time misconfiguration could impact correctness.  
    - **Version:** 2

---

#### 2.4 Image Handling

- **Overview:**  
Downloads the image from the RSS feed’s enclosure URL and uploads it to BlueSky to obtain a reference for embedding in the post.

- **Nodes Involved:**  
  - Download image  
  - Upload image

- **Node Details:**  
  - **Download image**  
    - **Type:** HTTP Request  
    - **Role:** Downloads the feed item’s image (binary file) from the enclosure URL.  
    - **Configuration:** URL dynamically set from the RSS Feed Trigger item JSON path `enclosure.url`. Response format is set to file (binary).  
    - **Inputs:** Receives current datetime node output.  
    - **Outputs:** Passes binary image data to "Upload image".  
    - **Edge cases:** If no image URL is available, node may fail or output empty data. Network errors or invalid URLs can cause download failure.  
    - **Version:** 4.2  
    - **Notes:** User can override image by adding a static URL here if feed lacks images.  

  - **Upload image**  
    - **Type:** HTTP Request  
    - **Role:** Uploads the downloaded image binary to BlueSky's blob storage endpoint, obtaining a blob reference for embedding.  
    - **Configuration:**  
      - URL: BlueSky upload blob endpoint.  
      - Method: POST  
      - Header: Authorization Bearer token from "Create Session" node's `accessJwt`.  
      - Content-Type: taken dynamically from the downloaded image's MIME type (`enclosure.type`).  
      - Sends binary data from previous node's output.  
    - **Inputs:** Receives binary image data from "Download image".  
    - **Outputs:** Passes uploaded image metadata (blob reference, MIME type, size) to "Create Post".  
    - **Edge cases:** Authorization errors, timeouts, or upload rejection by BlueSky. Incorrect content-type may cause upload failure.  
    - **Version:** 4.1

---

#### 2.5 Post Creation

- **Overview:**  
Builds and sends a request to BlueSky API to create a new post incorporating the RSS feed content, link, title, description, and embedded image blob.

- **Nodes Involved:**  
  - Create Post

- **Node Details:**  
  - **Create Post**  
    - **Type:** HTTP Request  
    - **Role:** Sends a POST request to BlueSky’s `createRecord` API to publish a feed post.  
    - **Configuration:**  
      - URL: BlueSky createRecord endpoint.  
      - Method: POST  
      - Authorization header uses Bearer token from session node.  
      - JSON body dynamically composed with:  
        - `repo`: User's DID from session node.  
        - `collection`: Fixed `"app.bsky.feed.post"`.  
        - `record`:  
          - `text`: Feed item content snippet (up to 200 chars), stringified JSON from `content:encodedSnippet`.  
          - `$type`: `"app.bsky.feed.post"`.  
          - `embed`: External link embedding with:  
            - `uri`: feed item URL (`link`).  
            - `title`: feed item title (`lintitlek`, possibly a typo).  
            - `description`: feed item snippet (`contentSnippet`).  
            - `thumb`: Blob metadata from uploaded image (type, ref, mimeType, size).  
          - `createdAt`: Current datetime from "Get current datetime".  
          - `langs`: `["es-ES"]` hardcoded language array.  
    - **Inputs:** Receives blob metadata from "Upload image".  
    - **Outputs:** Finalizes workflow.  
    - **Edge cases:** API errors from BlueSky (bad token, rate limits, malformed JSON), missing or malformed feed data, embedding failures if no image uploaded.  
    - **Version:** 4.1  
    - **Customization:** Users can modify the `text` field JSON to change post content (e.g., only title).  

---

### 3. Summary Table

| Node Name         | Node Type                | Functional Role                  | Input Node(s)          | Output Node(s)       | Sticky Note                                                                                   |
|-------------------|--------------------------|--------------------------------|-----------------------|----------------------|----------------------------------------------------------------------------------------------|
| RSS Feed Trigger   | RSS Feed Read Trigger     | Triggers workflow on new feed items | None                  | Create Session       |                                                                                              |
| Create Session    | HTTP Request             | Authenticates to BlueSky API   | RSS Feed Trigger       | Get current datetime  | ### Configure your credentials Create [an app password](https://bsky.app/settings/app-passwords) first |
| Get current datetime | DateTime                | Provides current date/time      | Create Session         | Download image        |                                                                                              |
| Download image    | HTTP Request             | Downloads feed item image       | Get current datetime   | Upload image          | ### Image preview By default retrieved from the feed, but you can configure a custom one here from an URL |
| Upload image      | HTTP Request             | Uploads image to BlueSky blob storage | Download image         | Create Post           |                                                                                              |
| Create Post       | HTTP Request             | Creates BlueSky post with feed content and image embed | Upload image           | None                 | ### Customize the text You can customize the message text here                              |
| Sticky Note       | Sticky Note              | Instructional note              | None                  | None                 | ### Configure your credentials Create [an app password](https://bsky.app/settings/app-passwords) first |
| Sticky Note1      | Sticky Note              | Instructional note              | None                  | None                 | ### Customize the text You can customize the message text here                              |
| Sticky Note2      | Sticky Note              | Instructional note              | None                  | None                 | ### Image preview By default retrieved from the feed, but you can configure a custom one here from an URL |
| Sticky Note3      | Sticky Note              | Workflow description note       | None                  | None                 | ## Post new RSS feed items as BlueSky posts This will create a BlueSky post with each new RSS feed item, including the feed title, post image, link and content (up to 200 characters) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create RSS Feed Trigger node**  
   - Type: RSS Feed Read Trigger  
   - Set `feedUrl` to your RSS feed URL.  
   - Poll interval: every minute.  

2. **Create HTTP Request node for BlueSky session ("Create Session")**  
   - Method: POST  
   - URL: `https://bsky.social/xrpc/com.atproto.server.createSession`  
   - Body parameters (JSON):  
     - `identifier`: your BlueSky username  
     - `password`: your BlueSky app password (generate via https://bsky.app/settings/app-passwords)  
   - Connect output of RSS Feed Trigger to this node’s input.  

3. **Create DateTime node ("Get current datetime")**  
   - Default settings.  
   - Connect output of "Create Session" to this node.  

4. **Create HTTP Request node to download image ("Download image")**  
   - Method: GET  
   - URL: Expression referencing feed item image URL: `{{$node["RSS Feed Trigger"].item.json["enclosure"]["url"]}}`  
   - Response format: File (binary)  
   - Connect output of "Get current datetime" to this node.  

5. **Create HTTP Request node to upload image to BlueSky ("Upload image")**  
   - Method: POST  
   - URL: `https://bsky.social/xrpc/com.atproto.repo.uploadBlob`  
   - Content-Type: Expression from downloaded image MIME type: `{{$json["enclosure"]["type"]}}`  
   - Authorization header: `Bearer {{$item(0).$node["Create Session"].json["accessJwt"]}}`  
   - Send binary data field name: `data` (the downloaded image)  
   - Connect output of "Download image" to this node.  

6. **Create HTTP Request node to create BlueSky post ("Create Post")**  
   - Method: POST  
   - URL: `https://bsky.social/xrpc/com.atproto.repo.createRecord`  
   - Authorization header: `Bearer {{$item(0).$node["Create Session"].json["accessJwt"]}}`  
   - Body Type: JSON  
   - Body content: Compose JSON with fields:  
     - `repo`: `{{$node["Create Session"].json["did"]}}`  
     - `collection`: `"app.bsky.feed.post"`  
     - `record`:  
       - `text`: `{{ JSON.stringify($node["RSS Feed Trigger"].json["content:encodedSnippet"]) }}` (customize as needed)  
       - `$type`: `"app.bsky.feed.post"`  
       - `embed`: External link object with:  
         - `uri`: feed item link  
         - `title`: feed item title (check correct JSON path, e.g. `title`)  
         - `description`: feed item snippet  
         - `thumb`: Blob metadata from "Upload image" output  
       - `createdAt`: current datetime from "Get current datetime" node  
       - `langs`: `["es-ES"]` (language code, adjust if needed)  
   - Connect output of "Upload image" to this node.  

7. **Connect all nodes according to the sequence:**  
   RSS Feed Trigger → Create Session → Get current datetime → Download image → Upload image → Create Post  

8. **Credentials Setup:**  
   - BlueSky authentication requires user credentials (username and app password) entered in "Create Session" node body parameters.  
   - The workflow obtains and uses the `accessJwt` token dynamically for authorization headers in image upload and post creation requests.  

9. **Customization Tips:**  
   - Modify the `text` field in "Create Post" node to change post content format (e.g., just the title or combined fields).  
   - If RSS feed lacks images, replace the URL in the "Download image" node with a static image URL.  
   - Adjust the language code in the post JSON if posting in a language other than Spanish (`es-ES`).  

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                     |
|----------------------------------------------------------------------------------------------------|-------------------------------------------------------------------|
| Generate a BlueSky app password to use for authentication: https://bsky.app/settings/app-passwords | Credential setup for BlueSky API authentication                    |
| Workflow description and purpose: Automates posting from RSS feed to BlueSky timeline.             | Workflow title and description node sticky note                   |
| Customize message text in the "Create Post" node JSON body to tailor post content.                 | Sticky Note on "Create Post" node                                  |
| By default, images are retrieved from the RSS feed enclosure; a custom image URL can be used instead. | Sticky Note on "Download image" node                              |

---

This document enables full understanding, reproduction, and customization of the "Post new RSS feed items as BlueSky posts" workflow, including potential failure points and configuration requirements.
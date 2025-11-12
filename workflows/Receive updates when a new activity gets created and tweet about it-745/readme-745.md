Receive updates when a new activity gets created and tweet about it

https://n8nworkflows.xyz/workflows/receive-updates-when-a-new-activity-gets-created-and-tweet-about-it-745


# Receive updates when a new activity gets created and tweet about it

### 1. Workflow Overview

This workflow automates the process of receiving notifications whenever a new activity is created on Strava and subsequently tweets about that activity on Twitter. It is suitable for users who want to publicly share their physical activity updates automatically without manual intervention.

Logical blocks:

- **1.1 Input Reception:** Listening for new activity creation events from Strava via a webhook trigger.
- **1.2 Social Media Posting:** Composing and sending a tweet with activity details to Twitter.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block listens for new activity creation events on Strava using a webhook trigger node. When an activity is created, it captures activity details such as distance and name.

- **Nodes Involved:**  
  - Strava Trigger

- **Node Details:**  
  - **Node Name:** Strava Trigger  
  - **Type:** Strava Trigger (Webhook Trigger Node)  
  - **Configuration:**  
    - Listens specifically for `create` events on the `activity` object in Strava.  
    - Uses OAuth2 credentials for Strava API authentication.  
    - No additional options configured.  
  - **Key Expressions/Variables:**  
    - Outputs JSON data with an `object_data` field containing activity details, e.g., `distance` and `name`.  
  - **Input Connections:** None (starting trigger node).  
  - **Output Connections:** Connects to the Twitter node.  
  - **Version Requirements:** Compatible with n8n version supporting Strava Trigger node v1.  
  - **Potential Failures:**  
    - OAuth2 token expiration or invalid credentials may cause authentication failures.  
    - Webhook setup issues on Strava’s side could prevent event delivery.  
    - Network timeouts or Strava API outages.  
  - **Sub-Workflow:** None.

#### 1.2 Social Media Posting

- **Overview:**  
  This block receives the activity data from Strava and posts a formatted tweet summarizing the activity details.

- **Nodes Involved:**  
  - Twitter

- **Node Details:**  
  - **Node Name:** Twitter  
  - **Type:** Twitter node (OAuth1 API integration)  
  - **Configuration:**  
    - The `text` parameter uses an expression to dynamically create the tweet content:  
      `"I ran {{$node["Strava Trigger"].json["object_data"]["distance"]}} meters and completed my {{$node["Strava Trigger"].json["object_data"]["name"]}}!"`  
    - No additional tweet fields configured (e.g., no media or hashtags).  
    - Uses OAuth1 credentials specific to a Twitter account.  
  - **Key Expressions/Variables:**  
    - Pulls distance and activity name from the Strava Trigger node output.  
  - **Input Connections:** Receives input from the Strava Trigger node.  
  - **Output Connections:** None (end node).  
  - **Version Requirements:** Compatible with n8n version supporting Twitter node v1.  
  - **Potential Failures:**  
    - OAuth1 credential expiration or misconfiguration.  
    - Twitter API rate limits or outages.  
    - Expression evaluation errors if expected fields are missing in input data.  
  - **Sub-Workflow:** None.

---

### 3. Summary Table

| Node Name      | Node Type          | Functional Role           | Input Node(s)     | Output Node(s)  | Sticky Note                                       |
|----------------|--------------------|--------------------------|-------------------|-----------------|--------------------------------------------------|
| Strava Trigger | Strava Trigger     | Receive new Strava activity events | None              | Twitter         |                                                  |
| Twitter        | Twitter            | Post tweet about new activity | Strava Trigger    | None            |                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Strava Trigger node:**
   - Set node name to `Strava Trigger`.
   - Select event: `create`.
   - Select object: `activity`.
   - Configure credentials:  
     - Create or select existing OAuth2 credentials for Strava API access.  
     - Ensure OAuth2 client ID, secret, and redirect URLs are correctly set up in Strava developer portal and n8n.  
   - Position node on the canvas.

3. **Add a Twitter node:**
   - Set node name to `Twitter`.
   - Configure the `Text` field with the expression:  
     `I ran {{$node["Strava Trigger"].json["object_data"]["distance"]}} meters and completed my {{$node["Strava Trigger"].json["object_data"]["name"]}}!`  
   - Configure credentials:  
     - Create or select existing OAuth1 credentials for Twitter API.  
     - Ensure the Twitter app and access tokens are properly set up with write permissions.  
   - Position node on the canvas to the right of the Strava Trigger node.

4. **Connect the Strava Trigger node output to the Twitter node input.**

5. **Activate the workflow** to start listening for new Strava activity events and posting tweets when triggered.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                                             |
|-----------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------|
| For OAuth2 Strava credentials setup, refer to Strava API documentation and ensure correct webhook registration in Strava. | https://developers.strava.com/docs/webhooks/                                                                               |
| For Twitter API OAuth1 setup and tweet posting guidelines, visit Twitter Developer Platform docs.    | https://developer.twitter.com/en/docs/authentication/oauth-1-0a                                                           |
| This workflow requires network access to both Strava and Twitter APIs and proper credential permissions. |                                                                                                                             |
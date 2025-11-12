Weekly Website Link Checker with Slack Alerts for Broken URLs

https://n8nworkflows.xyz/workflows/weekly-website-link-checker-with-slack-alerts-for-broken-urls-6647


# Weekly Website Link Checker with Slack Alerts for Broken URLs

### 1. Workflow Overview

This workflow automates a weekly scan of a website’s links to identify broken URLs (HTTP 404 errors) and sends alerts via Slack to notify the development team. It is designed to maintain website health by regularly verifying link integrity and proactively reporting issues.

The workflow is composed of three main logical blocks:

- **1.1 Scheduled Trigger & Link Retrieval:** Initiates the workflow weekly and fetches all URLs from the website sitemap or API.
- **1.2 Broken Link Detection & Processing:** Filters out broken links from the retrieved URLs and compiles them into a list.
- **1.3 Alert Dispatch & Non-Critical Handling:** Sends a Slack alert with the broken URL list and gracefully handles valid links with no further action.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger & Link Retrieval

- **Overview:**  
  This block triggers the workflow on a weekly schedule and fetches all links from the website’s sitemap or API endpoint for scanning.

- **Nodes Involved:**  
  - Weekly Cron Trigger  
  - Scan Blog with HTTP

- **Node Details:**

  - **Weekly Cron Trigger**  
    - Type: Cron Trigger  
    - Role: Schedules the workflow to run every Tuesday at midnight (default settings implied)  
    - Configuration: Default weekly schedule (no parameters explicitly set in JSON, but implied by node name and note)  
    - Input: None (trigger node)  
    - Output: Triggers next node  
    - Edge Cases: Workflow runs only if n8n instance is active; time zone mismatches might affect schedule  
    - Notes: Triggers workflow weekly for routine maintenance scanning

  - **Scan Blog with HTTP**  
    - Type: HTTP Request  
    - Role: Fetches the sitemap.xml from the blog URL to retrieve all website links for scanning  
    - Configuration:  
      - URL: `https://yourblog.com/sitemap.xml` (placeholder, must be updated to actual sitemap URL)  
      - Method: GET (default)  
      - No additional options or authentication configured  
    - Input: Trigger from Cron node  
    - Output: HTTP response containing the sitemap or link data for further processing  
    - Edge Cases:  
      - If the sitemap URL is unreachable or returns error, workflow fails or returns empty data  
      - Sitemap format must be parseable or compatible with downstream logic  
    - Notes: This node’s output is expected to be a list or collection of URLs with status info for filtering

#### 2.2 Broken Link Detection & Processing

- **Overview:**  
  This block identifies broken links by checking HTTP status codes and compiles these into a structured list for alerting.

- **Nodes Involved:**  
  - Filter Broken Links (IF node)  
  - Create Broken Links List (Function node)  
  - No Action for Valid Links (NoOp node)

- **Node Details:**

  - **Filter Broken Links**  
    - Type: IF node  
    - Role: Filters links based on HTTP status code to separate broken (404) from valid links  
    - Configuration:  
      - Condition: Number equals 404 for the property `status` from the HTTP node response  
      - Expression: `{{$node['Scan Blog with HTTP'].json['status']}} == 404`  
    - Input: Output from HTTP Request node (each link’s HTTP status)  
    - Output:  
      - True branch: Links with status 404 (broken)  
      - False branch: Valid links (non-404)  
    - Edge Cases:  
      - If status is missing or malformed, filtering may fail or misclassify links  
      - Assumes status code is numeric and accessible at `json.status`  
    - Notes: Critical for isolating broken URLs for alerting

  - **Create Broken Links List**  
    - Type: Function  
    - Role: Processes the filtered broken links and creates an array of their URLs for reporting  
    - Configuration:  
      - JavaScript code loops through all input items, extracts URLs where `status === 404`  
      - Returns one item with key `brokenLinks` containing an array of broken URLs  
    - Key Expression:  
      ```js
      const brokenLinks = [];
      for (let item of $input.all()) {
        if (item.json.status === 404) {
          brokenLinks.push(item.json.url);
        }
      }
      return [{ json: { brokenLinks: brokenLinks } }];
      ```  
    - Input: True branch from Filter Broken Links  
    - Output: Single JSON object with `brokenLinks` array  
    - Edge Cases:  
      - If no broken links, returns empty array  
      - Assumes each item has `url` and `status` properties  
      - Failure if input data is malformed or missing expected fields  
    - Notes: Prepares data for Slack notification

  - **No Action for Valid Links**  
    - Type: NoOp (No Operation)  
    - Role: Placeholder node for valid links where no further processing is required  
    - Configuration: None  
    - Input: False branch from Filter Broken Links  
    - Output: None (ends path)  
    - Edge Cases: None  
    - Notes: Keeps workflow logic clean; explicitly shows no action on valid links

#### 2.3 Alert Dispatch & Non-Critical Handling

- **Overview:**  
  This block sends a Slack message alerting the development team about broken links found during the scan.

- **Nodes Involved:**  
  - Send Slack Alert

- **Node Details:**

  - **Send Slack Alert**  
    - Type: Slack node  
    - Role: Sends a message to a specified Slack channel with a report of broken links  
    - Configuration:  
      - Channel: `dev-team` (Slack channel name)  
      - Message Text:  
        ```
        🚨 Broken Links Report:
        Found {{ $node['Create Broken Links List'].json['brokenLinks'].length }} broken links:
        {{ $node['Create Broken Links List'].json['brokenLinks'].join('\n') }}
        ```  
      - Uses dynamic expressions to include count and URLs in message  
      - Credentials: Slack API OAuth2 credential named "Slack account - test"  
    - Input: Output of Create Broken Links List (containing broken URLs)  
    - Output: Message posted to Slack channel  
    - Edge Cases:  
      - Slack API errors (auth, rate limits, invalid channel)  
      - Empty brokenLinks array results in message with zero count and no links  
    - Notes: Critical for alerting on broken URLs

---

### 3. Summary Table

| Node Name                | Node Type       | Functional Role                           | Input Node(s)           | Output Node(s)               | Sticky Note                                                                                               |
|--------------------------|-----------------|-----------------------------------------|------------------------|-----------------------------|---------------------------------------------------------------------------------------------------------|
| Weekly Cron Trigger       | Cron Trigger    | Triggers workflow weekly                 | None                   | Scan Blog with HTTP          | This node triggers the workflow every Tuesday at midnight to perform a weekly scan of the blog.         |
| Scan Blog with HTTP       | HTTP Request    | Fetches all links from blog sitemap      | Weekly Cron Trigger    | Filter Broken Links          | This node sends an HTTP request to the blog sitemap or API (e.g., Screaming Frog) to fetch all links.    |
| Filter Broken Links       | IF              | Filters broken links (404)                | Scan Blog with HTTP    | Create Broken Links List, No Action for Valid Links | This IF node filters links with a 404 status code to identify broken links.                              |
| Create Broken Links List  | Function        | Compiles broken URLs into a list          | Filter Broken Links (true branch) | Send Slack Alert           | This node processes the filtered data and creates a list of broken URLs for reporting.                   |
| Send Slack Alert          | Slack           | Sends Slack notification with broken links | Create Broken Links List | None                      | This node sends a Slack message to the 'dev-team' channel with the list of broken URLs.                  |
| No Action for Valid Links | NoOp            | Placeholder for valid links with no action | Filter Broken Links (false branch) | None                    | This node is a placeholder for valid links (non-404), where no action is taken.                          |
| Sticky Note              | Sticky Note     | Documentation note                       | None                   | None                        | ## System Architecture<br>- **Link Checking Pipeline**:<br>  - **Weekly Cron Trigger**: Schedules the workflow to run weekly.<br>  - **Scan Blog with HTTP**: Performs HTTP GET requests to check website links.<br>- **Alert and Tracking Flow**:<br>  - **Filter Broken Links**: Identifies and separates broken links.<br>  - **Send Slack Alert**: Notifies the team via Slack about broken URLs.<br>  - **Create Broken Links List**: Compiles a list of broken links.<br>- **Non-Critical Handling**:<br>  - **No Action for Valid Links**: Skips valid links with no further action. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**  
   - Add a **Cron** node named `Weekly Cron Trigger`.  
   - Configure it to trigger **every Tuesday at midnight** (e.g., set Day of Week to 2, Hour to 0, Minute to 0).  
   - This node will start the workflow weekly.

2. **Create HTTP Request Node to Fetch Links:**  
   - Add an **HTTP Request** node named `Scan Blog with HTTP`.  
   - Set the **URL** parameter to your blog sitemap URL, e.g., `https://yourblog.com/sitemap.xml`.  
   - Use GET method (default).  
   - No authentication or special headers are needed unless your site requires them.  
   - Connect the output of `Weekly Cron Trigger` to this node.

3. **Add IF Node to Filter Broken Links:**  
   - Add an **IF** node named `Filter Broken Links`.  
   - Configure condition: Check if the numeric value of the HTTP status code equals 404.  
     - Use expression: `{{$node["Scan Blog with HTTP"].json["status"]}}` equals `404`.  
   - Connect the output of `Scan Blog with HTTP` to this node.

4. **Add Function Node to Compile Broken Links:**  
   - Add a **Function** node named `Create Broken Links List`.  
   - Paste the following JavaScript code in the Function Code field:  
     ```js
     const brokenLinks = [];
     for (let item of $input.all()) {
       if (item.json.status === 404) {
         brokenLinks.push(item.json.url);
       }
     }
     return [{ json: { brokenLinks: brokenLinks } }];
     ```  
   - Connect the **True** output of `Filter Broken Links` to this node.

5. **Add Slack Node to Send Alert:**  
   - Add a **Slack** node named `Send Slack Alert`.  
   - Configure the Slack credentials (OAuth2) for your Slack workspace.  
   - Set the **Channel** to `dev-team` or your desired alert channel.  
   - Set the **Text** field to the following expression to dynamically include broken links:  
     ```
     🚨 Broken Links Report:
     Found {{ $node['Create Broken Links List'].json['brokenLinks'].length }} broken links:
     {{ $node['Create Broken Links List'].json['brokenLinks'].join('\n') }}
     ```  
   - Connect the output of `Create Broken Links List` to this node.

6. **Add No Operation Node for Valid Links:**  
   - Add a **NoOp** node named `No Action for Valid Links`.  
   - Connect the **False** output of `Filter Broken Links` to this node.  
   - No further configuration is needed; it acts as a placeholder.

7. **Add a Sticky Note (Optional):**  
   - Add a **Sticky Note** node for documentation with the following content:  
     ```
     ## System Architecture
     - **Link Checking Pipeline**:
       - **Weekly Cron Trigger**: Schedules the workflow to run weekly.
       - **Scan Blog with HTTP**: Performs HTTP GET requests to check website links.
     - **Alert and Tracking Flow**:
       - **Filter Broken Links**: Identifies and separates broken links.
       - **Send Slack Alert**: Notifies the team via Slack about broken URLs.
       - **Create Broken Links List**: Compiles a list of broken links.
     - **Non-Critical Handling**:
       - **No Action for Valid Links**: Skips valid links with no further action.
     ```

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                   |
|-----------------------------------------------------------------------------------------------------|--------------------------------------------------|
| The workflow requires updating the `Scan Blog with HTTP` node URL to your actual sitemap or API.     | Replace `https://yourblog.com/sitemap.xml`       |
| Slack credentials must be configured with OAuth2 for the Slack workspace and channel used.           | Credential named "Slack account - test"           |
| Tested n8n version: v1.x (ensure compatibility with HTTP Request and Slack nodes)                    | Check n8n changelog for breaking changes          |
| For large sitemaps, consider pagination or batch processing to avoid timeouts or rate limits.        | Performance optimization                           |
| Slack API rate limits or network issues may cause alert failures; implement retry or error handling. | Operational considerations                         |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.
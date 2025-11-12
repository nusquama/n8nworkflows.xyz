Track Daily Product Hunt Launches with Website Verification in Google Sheets

https://n8nworkflows.xyz/workflows/track-daily-product-hunt-launches-with-website-verification-in-google-sheets-3588


# Track Daily Product Hunt Launches with Website Verification in Google Sheets

### 1. Workflow Overview

This workflow automates the daily tracking of new product launches on Product Hunt by fetching product details, verifying website URLs for redirections, and logging the consolidated data into a Google Sheet. It is designed for indie hackers, product managers, content curators, and anyone interested in monitoring daily Product Hunt launches.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger and Date Setup**: Initiates the workflow daily and sets the current date for API filtering.
- **1.2 Product Hunt API Query**: Retrieves the day’s Product Hunt posts using a GraphQL query.
- **1.3 Product Data Extraction**: Parses the API response to extract relevant product information.
- **1.4 Website Redirection Check**: Checks if each product’s website URL redirects to a different URL.
- **1.5 Data Merging**: Combines the extracted product info with the final resolved URLs.
- **1.6 Google Sheets Logging**: Appends or updates the consolidated data into a Google Sheet for record-keeping.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger and Date Setup

**Overview:**  
This block triggers the workflow once daily and sets the current date in ISO format to filter Product Hunt posts for that day.

**Nodes Involved:**  
- Daily Trigger1 (Schedule Trigger)  
- Set Date1 (Set Node)

**Node Details:**  

- **Daily Trigger1**  
  - *Type & Role:* Schedule Trigger node; initiates workflow execution daily.  
  - *Configuration:* Runs once every day (default interval).  
  - *Inputs/Outputs:* No input; outputs trigger to Set Date1.  
  - *Edge Cases:* Workflow will not run if n8n instance is offline; misconfigured schedule could cause no trigger.  

- **Set Date1**  
  - *Type & Role:* Set node; sets a variable `today` with the current date in `YYYY-MM-DD` format.  
  - *Configuration:* Uses expression `={{ new Date().toISOString().split('T')[0] }}` to get the date.  
  - *Inputs/Outputs:* Input from Daily Trigger1; output to Product Hunt API HTTP Request node.  
  - *Edge Cases:* Timezone differences could affect the date; ensure n8n server time aligns with desired timezone.

---

#### 1.2 Product Hunt API Query

**Overview:**  
Fetches the first 10 Product Hunt posts published on the current date using a GraphQL API call.

**Nodes Involved:**  
- Fetches today’s Product Hunt posts via API. (HTTP Request)

**Node Details:**  

- **Fetches today’s Product Hunt posts via API.**  
  - *Type & Role:* HTTP Request node; performs POST request to Product Hunt GraphQL API.  
  - *Configuration:*  
    - URL: `https://api.producthunt.com/v2/api/graphql`  
    - Method: POST  
    - Headers include `Authorization: Bearer YOUR_PRODUCT_HUNT_API_KEY` (replace with actual token) and a User-Agent string.  
    - Body: GraphQL query requesting first 10 posts posted after and before the current date (using the `today` variable).  
  - *Expressions:* Uses `{{ $node["Set Date1"].json["today"] }}` to dynamically insert the date.  
  - *Inputs/Outputs:* Input from Set Date1; output to Extracts Product Info node.  
  - *Edge Cases:*  
    - Invalid or expired API token causes authentication errors.  
    - API rate limits or downtime may cause failures.  
    - If no posts are found, subsequent nodes must handle empty data gracefully.  
  - *Sticky Note:* Contains a link to Product Hunt OAuth Token Guide for authentication setup.

---

#### 1.3 Product Data Extraction

**Overview:**  
Parses the API response to extract key product details: name, tagline, description, and website URL.

**Nodes Involved:**  
- Extracts Product Info (Code node)  
- Data 1 (product info) (Set node)

**Node Details:**  

- **Extracts Product Info**  
  - *Type & Role:* Code node; transforms raw API response into structured product data items.  
  - *Configuration:* JavaScript code maps through `data.posts.edges` to extract `name`, `tagline`, `description`, and `website` fields into separate JSON objects.  
  - *Inputs/Outputs:* Input from Product Hunt API HTTP Request; outputs two parallel streams: one to Resolve Website Redirection, one to Data 1 (product info).  
  - *Edge Cases:*  
    - If API response structure changes, code may fail.  
    - Empty or malformed data could cause errors.  

- **Data 1 (product info)**  
  - *Type & Role:* Set node; prepares product info fields (`name`, `tagline`, `description`) for merging.  
  - *Configuration:* Maps fields from input JSON to output JSON with the same keys.  
  - *Inputs/Outputs:* Input from Extracts Product Info; output to Merge Data node.  
  - *Edge Cases:* Missing fields in input JSON could result in empty outputs.

---

#### 1.4 Website Redirection Check

**Overview:**  
Checks if the product website URL redirects to another URL by sending an HTTP request without following redirects.

**Nodes Involved:**  
- Resolve Website Redirection (HTTP Request)  
- Data 2 (website url) (Set node)

**Node Details:**  

- **Resolve Website Redirection**  
  - *Type & Role:* HTTP Request node; sends a GET request to the product’s website URL with redirect following disabled.  
  - *Configuration:*  
    - URL: dynamically set to `{{ $json.website }}` from product info.  
    - Options: `followRedirect` and `followAllRedirects` set to false to capture redirect headers.  
    - Full response enabled to access headers.  
    - Ignores response codes to avoid errors on redirect status codes.  
  - *Inputs/Outputs:* Input from Extracts Product Info; output to Data 2 (website url).  
  - *Edge Cases:*  
    - Invalid or unreachable URLs cause request failures.  
    - HTTPS certificate errors are ignored (`allowUnauthorizedCerts: true`), which may pose security risks.  
    - Websites without redirects will have no `location` header.  

- **Data 2 (website url)**  
  - *Type & Role:* Set node; extracts the `location` header from the HTTP response and stores it as `next_url`.  
  - *Configuration:* Sets `next_url` to `{{$json["headers"]["location"]}}`.  
  - *Inputs/Outputs:* Input from Resolve Website Redirection; output to Merge Data node.  
  - *Edge Cases:* If no redirect occurs, `location` header is undefined or null.

---

#### 1.5 Data Merging

**Overview:**  
Combines the product information and the resolved redirect URLs into a single dataset, cleaning URLs by removing tracking parameters.

**Nodes Involved:**  
- Merge Data (Function node)

**Node Details:**  

- **Merge Data**  
  - *Type & Role:* Function node; merges arrays of product info and redirect info by index.  
  - *Configuration:*  
    - Retrieves items from two previous nodes (`Data 1 (product info)` and `Data 2 (website url)`).  
    - For each product, constructs an object with `name`, `tagline`, `description`, and `next_url`.  
    - Cleans `next_url` by removing `?ref=producthunt` query parameter if present.  
  - *Inputs/Outputs:* Inputs from Data 1 and Data 2 nodes; output to Google Sheets node.  
  - *Edge Cases:*  
    - Mismatched array lengths could cause missing data.  
    - Null or undefined redirect URLs handled gracefully by setting `next_url` to null.  
    - Logs errors if input data is missing or inaccessible.

---

#### 1.6 Google Sheets Logging

**Overview:**  
Appends or updates the merged product data into a Google Sheet for persistent storage and tracking.

**Nodes Involved:**  
- Appends all details (Google Sheets node)

**Node Details:**  

- **Appends all details**  
  - *Type & Role:* Google Sheets node; appends or updates rows in a specified sheet.  
  - *Configuration:*  
    - Operation: `appendOrUpdate` (adds new rows or updates existing ones based on matching column).  
    - Matching Column: `name` (to avoid duplicates).  
    - Columns mapped: `name`, `tagline`, `description`, `next_url`.  
    - Authentication: uses Google Service Account credentials.  
    - Spreadsheet ID and Sheet Name are set (example uses `gid=0` and `demo` placeholders).  
  - *Inputs/Outputs:* Input from Merge Data node; no output.  
  - *Edge Cases:*  
    - Incorrect Spreadsheet ID or Sheet Name causes errors.  
    - Authentication failures if credentials are invalid or expired.  
    - Data type mismatches if sheet columns do not match expected schema.  
  - *Sticky Note:* Contains a link to a YouTube guide on connecting Google Sheets in n8n.

---

### 3. Summary Table

| Node Name                               | Node Type             | Functional Role                     | Input Node(s)                          | Output Node(s)                        | Sticky Note                                                                                                           |
|----------------------------------------|-----------------------|-----------------------------------|--------------------------------------|-------------------------------------|-----------------------------------------------------------------------------------------------------------------------|
| Daily Trigger1                         | Schedule Trigger      | Initiates daily workflow           | -                                    | Set Date1                           |                                                                                                                       |
| Set Date1                             | Set                   | Sets current date for API query   | Daily Trigger1                       | Fetches today’s Product Hunt posts via API. |                                                                                                                       |
| Fetches today’s Product Hunt posts via API. | HTTP Request          | Queries Product Hunt API           | Set Date1                           | Extracts Product Info               | Contains link to Product Hunt OAuth Token Guide: https://api.producthunt.com/v2/docs/oauth_user_authentication/oauth_authorize_ask_for_access_grant_code_on_behalf_of_the_user |
| Extracts Product Info                 | Code                  | Parses API response to product data | Fetches today’s Product Hunt posts via API. | Resolve Website Redirection, Data 1 (product info) |                                                                                                                       |
| Data 1 (product info)                 | Set                   | Prepares product info for merging | Extracts Product Info               | Merge Data                         |                                                                                                                       |
| Resolve Website Redirection           | HTTP Request          | Checks website URL redirects      | Extracts Product Info               | Data 2 (website url)               |                                                                                                                       |
| Data 2 (website url)                  | Set                   | Extracts redirect URL from headers | Resolve Website Redirection         | Merge Data                         |                                                                                                                       |
| Merge Data                           | Function               | Combines product info and redirect URLs | Data 1 (product info), Data 2 (website url) | Appends all details               |                                                                                                                       |
| Appends all details                  | Google Sheets          | Appends or updates data in sheet  | Merge Data                         | -                                   | Contains link to Google Sheets connection guide: https://www.youtube.com/watch?v=pWGXlZBGu4k                            |
| Sticky Note16                       | Sticky Note            | Provides Product Hunt token guide | -                                    | -                                   | https://api.producthunt.com/v2/docs/oauth_user_authentication/oauth_authorize_ask_for_access_grant_code_on_behalf_of_the_user |
| Sticky Note17                       | Sticky Note            | Provides Google Sheets connection guide | -                                    | -                                   | https://www.youtube.com/watch?v=pWGXlZBGu4k                                                                            |
| Sticky Note18                       | Sticky Note            | Author introduction and portfolio links | -                                    | -                                   | Portfolio: https://ifeoluwaajetomobi.framer.website/, Behance: https://www.behance.net/ajetomoifeoluw                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Name: `Daily Trigger1`  
   - Set to run once every day (default interval).  

2. **Add a Set node**  
   - Name: `Set Date1`  
   - Add a string field named `today` with value: `={{ new Date().toISOString().split('T')[0] }}`  
   - Connect `Daily Trigger1` output to `Set Date1` input.  

3. **Add an HTTP Request node**  
   - Name: `Fetches today’s Product Hunt posts via API.`  
   - Method: POST  
   - URL: `https://api.producthunt.com/v2/api/graphql`  
   - Headers:  
     - `Authorization`: `Bearer YOUR_PRODUCT_HUNT_API_KEY` (replace with your actual token)  
     - `User-Agent`: `Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36`  
   - Body Parameters (GraphQL query):  
     ```
     query {
       posts(first: 10, postedAfter: "{{ $node["Set Date1"].json["today"] }}T00:00:00Z", postedBefore: "{{ $node["Set Date1"].json["today"] }}T23:59:59Z") {
         edges {
           node {
             name
             tagline
             description
             website
           }
           cursor
         }
         pageInfo {
           hasNextPage
           endCursor
         }
       }
     }
     ```  
   - Connect `Set Date1` output to this node input.  

4. **Add a Code node**  
   - Name: `Extracts Product Info`  
   - JavaScript code:  
     ```javascript
     return $json.data.posts.edges.map(edge => {
       return {
         json: {
           name: edge.node.name,
           tagline: edge.node.tagline,
           description: edge.node.description,
           website: edge.node.website
         }
       };
     });
     ```  
   - Connect output of HTTP Request node to this node.  

5. **Add an HTTP Request node**  
   - Name: `Resolve Website Redirection`  
   - Method: GET  
   - URL: `={{ $json.website }}` (dynamic from product info)  
   - Options:  
     - Enable `Full Response` to access headers  
     - Disable `Follow Redirect` and `Follow All Redirects` (set to false)  
     - Enable `Ignore Response Code`  
     - Allow Unauthorized Certificates (true)  
   - Connect output of `Extracts Product Info` to this node.  

6. **Add a Set node**  
   - Name: `Data 2 (website url)`  
   - Set a string field `next_url` with value: `={{ $json["headers"]["location"] }}`  
   - Enable option to keep only set fields.  
   - Connect output of `Resolve Website Redirection` to this node.  

7. **Add a Set node**  
   - Name: `Data 1 (product info)`  
   - Set string fields:  
     - `name`: `={{ $json.name }}`  
     - `tagline`: `={{ $json.tagline }}`  
     - `description`: `={{ $json.description }}`  
   - Enable option to keep only set fields.  
   - Connect output of `Extracts Product Info` to this node.  

8. **Add a Function node**  
   - Name: `Merge Data`  
   - JavaScript code:  
     ```javascript
     let productData = [];
     let redirectData = [];
     
     try {
       productData = $items("Data 1 (product info)");
     } catch (error) {
       console.log("Error fetching product data:", error.message);
     }
     
     try {
       redirectData = $items("Data 2 (website url)");
     } catch (error) {
       console.log("Error fetching redirect data:", error.message);
     }
     
     const mergedItems = [];
     
     for (let i = 0; i < productData.length; i++) {
       const product = productData[i].json;
       
       const mergedItem = {
         name: product.name,
         tagline: product.tagline,
         description: product.description,
         next_url: null
       };
       
       if (i < redirectData.length && redirectData[i] && redirectData[i].json) {
         let url = redirectData[i].json.next_url;
         if (url && url.includes('?ref=producthunt')) {
           url = url.replace('?ref=producthunt', '');
         }
         mergedItem.next_url = url;
       }
       
       mergedItems.push({ json: mergedItem });
     }
     
     return mergedItems;
     ```  
   - Connect outputs of `Data 1 (product info)` and `Data 2 (website url)` to this node.  

9. **Add a Google Sheets node**  
   - Name: `Appends all details`  
   - Operation: `appendOrUpdate`  
   - Authentication: Use Google Service Account credentials (set up in n8n Credentials)  
   - Spreadsheet ID: Enter your Google Sheet ID  
   - Sheet Name: Enter your sheet name (e.g., `Daily Launches`)  
   - Matching Column: `name`  
   - Columns to map:  
     - `name` → `name`  
     - `tagline` → `tagline`  
     - `description` → `description`  
     - `next_url` → `next_url`  
   - Connect output of `Merge Data` to this node.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                        | Context or Link                                                                                                  |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| To get your Product Hunt API token, follow the official OAuth guide: https://api.producthunt.com/v2/docs/oauth_user_authentication/oauth_authorize_ask_for_access_grant_code_on_behalf_of_the_user | Product Hunt API Authentication                                                                                  |
| To connect Google Sheets in n8n, watch this step-by-step video guide: https://www.youtube.com/watch?v=pWGXlZBGu4k                                                 | Google Sheets Credential Setup                                                                                   |
| Author: Ajetomobi Ifeoluwa – UI/UX Designer and Workflow Creator. Portfolio: https://ifeoluwaajetomobi.framer.website/, Behance: https://www.behance.net/ajetomoifeoluw | Author and Project Credits                                                                                        |
| Workflow fetches only the first 10 posts per day by default; this can be adjusted in the GraphQL query `first: 10`.                                               | Limitation / Customization                                                                                        |
| Consider adding notification nodes (Slack, Discord, Email) after Google Sheets to alert on new launches.                                                         | Suggested Enhancements                                                                                            |

---

This documentation fully describes the workflow structure, node configurations, and setup instructions, enabling advanced users or AI agents to understand, reproduce, and extend the workflow reliably.
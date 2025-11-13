Reliable Reddit Subreddit Search with OAuth2 API Authentication

https://n8nworkflows.xyz/workflows/reliable-reddit-subreddit-search-with-oauth2-api-authentication-7646


# Reliable Reddit Subreddit Search with OAuth2 API Authentication

### 1. Workflow Overview

This workflow enables reliable searching of Reddit subreddits using the Reddit OAuth2 API with authenticated requests. It addresses common issues caused by unauthenticated API calls by ensuring OAuth2 credentials are used, thus allowing access to subreddit data filtered by membership counts and search limits. The workflow is designed for integration as a sub-workflow, accepting search parameters and returning a cleaned, aggregated list of subreddits matching the criteria.

**Logical Blocks:**

- **1.1 Input Reception:** Receives input parameters (search query, member count limits, result limit) from an external workflow trigger.
- **1.2 API Request with OAuth2 Authentication:** Makes an authenticated API call to Reddit's subreddit search endpoint using OAuth2 credentials.
- **1.3 Data Extraction and Processing:** Splits the returned subreddit list, extracts and renames key fields, filters or aggregates data, and prepares the output.
- **1.4 Output Preparation:** Aggregates processed subreddit data into a clean output format.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
Receives input parameters as JSON from another workflow. Parameters include the search query, minimum and maximum member counts, and the maximum number of results to retrieve.

- **Nodes Involved:**  
  - When Executed by Another Workflow

- **Node Details:**

  - **When Executed by Another Workflow**  
    - *Type & Role:* Execute Workflow Trigger node; acts as entry point receiving input data from a parent workflow.  
    - *Configuration:* Uses a JSON example input with fields: Query (string), min_Members (number), max_members (number), and limit (number).  
    - *Expressions:* Input fields are accessed as `$json.Query`, `$json.min_Members`, etc. for downstream use.  
    - *Connections:* Outputs to "Get many Subreddit".  
    - *Edge Cases:* If the parent workflow sends incomplete or malformed JSON, downstream nodes may fail or return incomplete data. Proper input validation is recommended upstream.  
    - *Version:* 1.1

---

#### 2.2 API Request with OAuth2 Authentication

- **Overview:**  
Performs an authenticated GET request to Reddit's `/subreddits/search` endpoint using OAuth2 credentials to retrieve subreddit data matching the search query.

- **Nodes Involved:**  
  - Get many Subreddit

- **Node Details:**

  - **Get many Subreddit**  
    - *Type & Role:* HTTP Request node; queries Reddit API to fetch subreddit search results.  
    - *Configuration:*  
      - URL: `https://oauth.reddit.com/subreddits/search`  
      - Query parameters:  
        - `q`: search query from input JSON (`={{ $json.Query }}`)  
        - `limit`: number of subreddits to return (`={{ $json.limit }}`)  
      - Authentication: Uses "redditOAuth2Api" credential configured with OAuth2 tokens.  
    - *Expressions:* Query parameters dynamically set from workflow input JSON.  
    - *Connections:* Outputs to "Split Out" node.  
    - *Version:* 4.2  
    - *Edge Cases:*  
      - OAuth2 token expiration or invalid credentials may cause authorization errors (HTTP 401).  
      - API rate limits or network issues may cause request failures or timeouts.  
      - Query parameters missing or malformed may return empty or error responses.

---

#### 2.3 Data Extraction and Processing

- **Overview:**  
Processes the raw API response to extract individual subreddit data, map key fields to friendly names, and aggregate the results for output.

- **Nodes Involved:**  
  - Split Out  
  - Edit Fields  
  - Aggregate

- **Node Details:**

  - **Split Out**  
    - *Type & Role:* Split Out node; splits the array of subreddit objects in the response into separate items for individual processing.  
    - *Configuration:* Splits on `data.children` array from Reddit API response.  
    - *Connections:* Input from "Get many Subreddit", outputs to "Edit Fields".  
    - *Edge Cases:* If `data.children` is missing or empty, no items will be output.  
    - *Version:* 1

  - **Edit Fields**  
    - *Type & Role:* Set node; transforms and renames fields for clarity and easier downstream use.  
    - *Configuration:* Assigns:  
      - `Subreddit` = `{{$json.data.url}}` (subreddit URL path)  
      - `Description` = `{{$json.data.public_description}}` (public description text)  
      - `18+` = `{{$json.data.over18}}` (boolean indicating NSFW status)  
      - `Members` = `{{$json.data.subscribers}}` (number of subscribers)  
    - *Connections:* Input from "Split Out", outputs to "Aggregate".  
    - *Edge Cases:* Missing fields in API response might produce empty or null outputs for these variables.  
    - *Version:* 3.4

  - **Aggregate**  
    - *Type & Role:* Aggregate node; combines the processed subreddit items into a single aggregated data set for output.  
    - *Configuration:* Aggregates fields: Subreddit, Description, 18+, Members.  
    - *Connections:* Input from "Edit Fields".  
    - *Edge Cases:* Aggregation on empty input will result in empty output.  
    - *Version:* 1

---

#### 2.4 Output Preparation

- **Overview:**  
The final aggregated data is output from the workflow, intended to be passed back to the calling workflow or used downstream.

- **Nodes Involved:**  
  - None explicitly; the output is the result of the Aggregate node's output.

---

### 3. Summary Table

| Node Name                  | Node Type                   | Functional Role                          | Input Node(s)                 | Output Node(s)              | Sticky Note                                                                                                                       |
|----------------------------|-----------------------------|----------------------------------------|------------------------------|-----------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| When Executed by Another Workflow | Execute Workflow Trigger      | Entry point; receives input parameters  | -                            | Get many Subreddit           | ## Get many subreddit Alternative Node workflow<br>Do you keep getting an error response when using Get many subreddit node? Me too, so I made this Template to fix this issue, it turns out that reddit don't like when you request from their api without any Authorization Credentials. <br>## How to use?<br>1. Paste this workflow into your n8n<br>2. Replace Get Many Subreddit node with an Execute a SubWorkflow then select this workflow.<br>3. Add Queries,<br>Query - Your Keyword to finding multiple subreddits<br>min_members - Minimum amount of members you'd like to be returned<br>max_members - Max amount of members you'd like to be returned<br>limit - The number of subreddits you'd like to get. |
| Get many Subreddit          | HTTP Request                | Makes OAuth2 authenticated Reddit API call | When Executed by Another Workflow | Split Out                   | See above sticky note                                                                                                            |
| Split Out                  | Split Out                   | Splits API response array into items    | Get many Subreddit            | Edit Fields                 | # Data Processing<br>This section is made to process and retrieve the data needed and return the cleaned data                     |
| Edit Fields                | Set                         | Renames and extracts key subreddit fields | Split Out                    | Aggregate                   | See above sticky note                                                                                                            |
| Aggregate                  | Aggregate                   | Aggregates processed subreddit data     | Edit Fields                  | -                           | See above sticky note                                                                                                            |
| Sticky Note                | Sticky Note                 | Provides usage instructions               | -                            | -                           | ## Get many subreddit Alternative Node workflow<br>Do you keep getting an error response when using Get many subreddit node? Me too, so I made this Template to fix this issue, it turns out that reddit don't like when you request from their api without any Authorization Credentials. <br>## How to use?<br>1. Paste this workflow into your n8n<br>2. Replace Get Many Subreddit node with an Execute a SubWorkflow then select this workflow.<br>3. Add Queries,<br>Query - Your Keyword to finding multiple subreddits<br>min_members - Minimum amount of members you'd like to be returned<br>max_members - Max amount of members you'd like to be returned<br>limit - The number of subreddits you'd like to get. |
| Sticky Note1               | Sticky Note                 | Notes on data processing block            | -                            | -                           | # Data Processing<br>This section is made to process and retrieve the data needed and return the cleaned data                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add an "Execute Workflow Trigger" node:**
   - Name it: `When Executed by Another Workflow`.
   - Set Input Source to `JSON Example`.
   - Paste the example JSON:
     ```json
     {
       "Query": "RealEstateTechnology",
       "min_Members": 0,
       "max_members": 20000,
       "limit": 50
     }
     ```
   - This node will receive parameters when called as a sub-workflow.

3. **Add an "HTTP Request" node:**
   - Name it: `Get many Subreddit`.
   - Set the HTTP Method to `GET`.
   - Set the URL to `https://oauth.reddit.com/subreddits/search`.
   - Enable Query Parameters and add:
     - `q` = `={{ $json.Query }}` (expression to use the input query)
     - `limit` = `={{ $json.limit }}`
   - Under Authentication, select `Predefined Credential Type` and choose the Reddit OAuth2 credential (you must configure this credential in n8n beforehand with your Reddit app client ID, secret, and OAuth2 settings).
   - Connect output of `When Executed by Another Workflow` to this node.

4. **Add a "Split Out" node:**
   - Name it: `Split Out`.
   - Configure to split on the field `data.children`.
   - Connect output of `Get many Subreddit` to this node.

5. **Add a "Set" node:**
   - Name it: `Edit Fields`.
   - Add the following fields with values as expressions:
     - `Subreddit` = `={{ $json.data.url }}`
     - `Description` = `={{ $json.data.public_description }}`
     - `18+` = `={{ $json.data.over18 }}`
     - `Members` = `={{ $json.data.subscribers }}`
   - Connect output of `Split Out` to this node.

6. **Add an "Aggregate" node:**
   - Name it: `Aggregate`.
   - Configure to aggregate fields:
     - `Subreddit`
     - `Description`
     - `18+`
     - `Members`
   - Connect output of `Edit Fields` to this node.

7. **Save the workflow.**

8. **Optional Sticky Notes:**
   - Add Sticky Notes to provide instructions or explanations as in the original workflow:
     - One explaining the purpose and usage of the workflow and OAuth2 necessity.
     - One annotating the data processing section.

9. **Testing:**
   - Execute the workflow manually or call it from another workflow with appropriate input JSON.
   - Verify that the output contains aggregated subreddit data matching the query.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Context or Link                                                                                               |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| This workflow was created to fix errors encountered when calling Reddit's API without OAuth2 authentication, which is required for certain endpoints. It is recommended to use this workflow as a sub-workflow in place of direct unauthenticated calls.                                                                                                                                                                                                                                                                                                                                                                           | Sticky Note content within the workflow.                                                                     |
| Reddit OAuth2 API requires proper credential setup in n8n with client ID, client secret, and OAuth2 endpoints configured correctly. Ensure your Reddit app has the necessary permissions for subreddit search.                                                                                                                                                                                                                                                                                                                                                                                                             | Reddit developer documentation: https://www.reddit.com/dev/api/                                               |
| Input parameters include `Query` (keyword string), `min_Members` and `max_members` (for potential filtering, not currently implemented in this workflow but can be added), and `limit` (number of results).                                                                                                                                                                                                                                                                                                                                                                                                               | Defined in the input JSON example.                                                                            |
| This workflow outputs aggregated subreddit data fields including subreddit URL, description, NSFW flag, and subscriber count. It can be extended to filter subreddits by membership count using additional nodes.                                                                                                                                                                                                                                                                                                                                                                                                        | Workflow logic and potential extensions.                                                                     |
| For OAuth2 credential setup in n8n, consult the official n8n documentation on OAuth2 credential creation and Reddit API specifications for token endpoint and scopes.                                                                                                                                                                                                                                                                                                                                                                                                                                                  | https://docs.n8n.io/credentials/oauth2/ and https://www.reddit.com/dev/api/                                    |

---

**Disclaimer:** The provided text comes exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.
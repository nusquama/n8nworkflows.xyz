Scrape every url on the web without getting blocked by Anti-Bot technologies with Scrappey

https://n8nworkflows.xyz/workflows/scrape-every-url-on-the-web-without-getting-blocked-by-anti-bot-technologies-with-scrappey-2299


# Scrape every url on the web without getting blocked by Anti-Bot technologies with Scrappey

### 1. Workflow Overview

This workflow is designed to scrape any URL on the web without being blocked by common anti-bot technologies by leveraging the Scrappey API service. It is intended for users who need reliable web scraping without the usual issues related to bot detection and blocking.

The workflow is composed of three main logical blocks:

- **1.1 Trigger and Input Preparation:** Scheduling the workflow execution and preparing test input data (URLs).
- **1.2 Web Scraping API Call:** Making a POST request to the Scrappey API with the URL to scrape, including authentication.
- **1.3 Output Delivery:** Receiving the scraped website data for further processing or usage.

---

### 2. Block-by-Block Analysis

#### 2.1 Block: Trigger and Input Preparation

**Overview:**  
This block initiates the workflow on a scheduled interval and prepares the URL data for scraping. It currently uses a static test URL but is designed to accept dynamic input for production use.

**Nodes Involved:**  
- Schedule Trigger  
- Test Data  
- Sticky Note (Test Data explanation)

**Node Details:**

- **Schedule Trigger**  
  - *Type:* Trigger node (Schedule Trigger)  
  - *Role:* Starts the workflow execution automatically based on a time interval.  
  - *Configuration:* Runs on default interval (every minute by default, as indicated by an empty interval array).  
  - *Input/Output:* No input; outputs trigger signal to "Test Data" node.  
  - *Edge Cases:* Misconfiguration can cause unexpected runtime intervals or no execution.  
  - *Version:* 1.2  
  - *Sticky Note:* See "Sticky Note" node describing test data usage.

- **Test Data**  
  - *Type:* Set node  
  - *Role:* Provides static test input data containing a website name and URL to scrape.  
  - *Configuration:* Sets two string fields: `name` = "n8n", `url` = "https://n8n.io/".  
  - *Input/Output:* Receives trigger from Schedule Trigger; outputs data to the HTTP Request node.  
  - *Edge Cases:* Must be replaced or extended for production with dynamic or multiple URLs.  
  - *Version:* 3.3  
  - *Sticky Note:* "Using n8n.io as test url. For production use, you have to connect your data here."

- **Sticky Note (Test Data explanation)**  
  - *Type:* Sticky Note node  
  - *Role:* Documentation for the user, indicating that the test data is currently static and should be replaced for production.  
  - *Position:* Visually linked to the Test Data node.

#### 2.2 Block: Web Scraping API Call

**Overview:**  
This block sends the URL to be scraped to the Scrappey API service using an authenticated HTTP POST request and receives the scraped website data in response.

**Nodes Involved:**  
- Scrape website with Scrappey  
- Sticky Note1 (Scrappey API usage instructions)

**Node Details:**

- **Scrape website with Scrappey**  
  - *Type:* HTTP Request node  
  - *Role:* Sends a POST request to Scrappey’s API endpoint to scrape the provided URL.  
  - *Configuration:*  
    - URL: `https://publisher.scrappey.com/api/v1`  
    - Method: POST  
    - Body: Form parameters containing:  
      - `cmd` = "request.get" (command to perform a GET request on the target URL)  
      - `url` = expression to dynamically take the URL from input data `={{ $json.url }}`  
    - Query Parameters:  
      - `key` = "YOUR_API_KEY" (to be replaced with the user's actual Scrappey API key)  
    - Options: Redirects are followed automatically.  
  - *Input/Output:* Receives input from "Test Data" node; outputs the scraped website data.  
  - *Edge Cases:*  
    - API authentication failure if `YOUR_API_KEY` is not replaced or invalid.  
    - Network or timeout errors.  
    - Unexpected API response format.  
    - Anti-bot detection may still occur if Scrappey’s service is down or limited.  
  - *Version:* 4.2  
  - *Sticky Note:* "Using Scrappey's API to scrape every website. Don't get blocked again by anti-bot technologies while scraping the web. Setup: Replace YOUR_API_KEY with your Scrappey API key."

- **Sticky Note1 (Scrappey API usage instructions)**  
  - *Type:* Sticky Note node  
  - *Role:* Provides the user instructions and important setup notes for using the Scrappey API key.  
  - *Content Includes:* Link to [Scrappey registration](https://scrappey.com/?ref=n8n) for free API key signup.

---

### 3. Summary Table

| Node Name                 | Node Type             | Functional Role                 | Input Node(s)        | Output Node(s)               | Sticky Note                                                                                               |
|---------------------------|-----------------------|--------------------------------|----------------------|-----------------------------|-----------------------------------------------------------------------------------------------------------|
| Schedule Trigger          | Schedule Trigger      | Initiate workflow periodically | None                 | Test Data                   |                                                                                                           |
| Test Data                 | Set                   | Provide static test URL data    | Schedule Trigger     | Scrape website with Scrappey | Using n8n.io as test url. For production use, you have to connect your data here.                          |
| Scrape website with Scrappey | HTTP Request         | Call Scrappey API to scrape URL | Test Data            | None                        | Using Scrappey's API to scrape every website. Don't get blocked again by anti-bot technologies while scraping the web. Setup: Replace YOUR_API_KEY with your Scrappey API key. https://scrappey.com/?ref=n8n |
| Sticky Note               | Sticky Note           | Document test data usage        | None                 | None                        | ## Test Data \n\nUsing n8n.io as test url.\n\nFor production use, you have to connect your data here.     |
| Sticky Note1              | Sticky Note           | Document Scrappey API usage     | None                 | None                        | ## Web Scraping \n\nUsing **Scrappey's** API to scrape every website.\n\nDon't get blocked again by anti-bot technologies while scraping the web.\n\n**Setup:**\nReplace YOUR_API_KEY with [your Scrappey API key.](https://scrappey.com/?ref=n8n) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Configuration: Use default interval (runs every minute or customize as needed).

2. **Create the Test Data node:**  
   - Type: Set  
   - Connect its input to the Schedule Trigger node output.  
   - Configuration:  
     - Add two fields:  
       - `name`: string, value "n8n"  
       - `url`: string, value "https://n8n.io/"  
   - Note: Replace or extend this node with your production URLs or data sources.

3. **Create the HTTP Request node to call Scrappey:**  
   - Name it "Scrape website with Scrappey".  
   - Connect its input to the Test Data node output.  
   - Configuration:  
     - HTTP Method: POST  
     - URL: `https://publisher.scrappey.com/api/v1`  
     - Query Parameters: Add `key` with value `"YOUR_API_KEY"` — replace this with your actual Scrappey API key.  
     - Body Parameters (Form):  
       - `cmd`: `request.get`  
       - `url`: use expression `{{$json["url"]}}` to dynamically get the URL from input data.  
     - Options: Enable automatic redirects.  
   - Credentials: None required beyond the API key in query parameters.  
   - Important: Replace `"YOUR_API_KEY"` with your Scrappey API key obtained from https://scrappey.com/?ref=n8n.

4. **Add Sticky Notes for documentation (optional but recommended):**  
   - One near the Test Data node explaining the purpose of the test data and replacement instructions.  
   - One near the HTTP Request node explaining the Scrappey API usage and setup instructions including the link to obtain the API key.

5. **Save and activate your workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                   |
|-----------------------------------------------------------------------------------------------------|--------------------------------------------------|
| Replace `YOUR_API_KEY` in the HTTP Request node with your Scrappey API key to enable scraping.       | https://scrappey.com/?ref=n8n                     |
| This workflow uses Scrappey's API to avoid anti-bot blocks common in web scraping.                    | Scrappey official website                          |
| For production, replace the static test URL with dynamic or batch inputs from databases or other nodes. | Workflow scalability advice                        |
| Scrappey API supports various commands; this workflow uses `request.get` to fetch webpage content.   | Scrappey API documentation (refer to Scrappey site) |

---

This documentation fully describes the workflow structure, node configurations, potential issues, and instructions for reproduction, enabling both technical users and automation agents to understand, operate, and modify the workflow confidently.
Scrape Google Events data to Google Sheets via SerpApi

https://n8nworkflows.xyz/workflows/scrape-google-events-data-to-google-sheets-via-serpapi-6105


# Scrape Google Events data to Google Sheets via SerpApi

### 1. Workflow Overview

This workflow automates the retrieval of event data from Google Events via the SerpApi service and appends the extracted information into a Google Sheets document for analysis or reporting. It is designed for users who want to monitor or gather event listings based on specific queries (e.g., location-based events) without manually scraping or copying data.

Logical blocks of the workflow include:

- **1.1 Input Reception:** Manual trigger to start the workflow and setting up search parameters such as query string, total events to fetch, and pagination start.
- **1.2 API Request & Pagination:** Calls SerpApi's Google Events engine with defined query parameters, handling pagination with rate limiting.
- **1.3 Data Processing:** Flattens and restructures the nested event data returned by the API, enriching each event with the original search query.
- **1.4 Data Output:** Appends the processed event data to a specified Google Sheets document and tab, organizing the fields for easy analysis.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block initializes the workflow by providing a manual trigger for on-demand execution and sets the search parameters including the event query, the total number of events to retrieve, and the pagination starting point.

- **Nodes Involved:**  
  - Manual Trigger  
  - Set Search Parameters  

- **Node Details:**  

  - **Manual Trigger**  
    - *Type:* Manual trigger node  
    - *Role:* Starts the workflow manually when the user desires to run a fresh data fetch.  
    - *Configuration:* No parameters required; simple trigger interface.  
    - *Input/Output:* No input; output connects to "Set Search Parameters".  
    - *Edge Cases:* User forgetting to trigger or triggering multiple times rapidly may cause duplicate data unless controlled downstream.

  - **Set Search Parameters**  
    - *Type:* Set node  
    - *Role:* Defines the search query and pagination parameters used for the API request.  
    - *Configuration:* Sets three JSON properties:  
      - `query`: "Events in Texas" (search topic/location)  
      - `total_events`: 10 (total number of events to fetch)  
      - `start`: 0 (pagination start index)  
    - *Key Expressions:* Static JSON output with defined keys.  
    - *Input/Output:* Input from Manual Trigger; output to "SerpApi Events Request".  
    - *Edge Cases:* Parameters may need adjustment for different locations/topics or event counts; improper values could lead to zero or excessive API calls.

#### 2.2 API Request & Pagination

- **Overview:**  
  This block calls the SerpApi Google Events engine API using the parameters set earlier, supports pagination with a 2-second delay between requests, and limits the requests based on the total events desired.

- **Nodes Involved:**  
  - SerpApi Events Request  

- **Node Details:**  

  - **SerpApi Events Request**  
    - *Type:* HTTP Request node  
    - *Role:* Sends HTTP GET requests to SerpApi to fetch Google Events data.  
    - *Configuration:*  
      - URL: `https://serpapi.com/search`  
      - Query parameters:  
        - `engine`: "google_events"  
        - `q`: dynamic from previous node, i.e., the search query  
        - `hl`: "en" (language)  
        - `gl`: "us" (country)  
      - Authentication: HTTP Query Auth with SerpApi credentials (requires valid API key).  
      - Pagination settings:  
        - Pagination parameter: `start` (offset), dynamically set from `start` value.  
        - Maximum requests calculated as `total_events / 10` (10 events per page).  
        - 2000ms delay between requests to respect rate limits.  
    - *Input/Output:* Receives search parameters as input; outputs JSON responses containing event results.  
    - *Edge Cases:*  
      - API key or credential failure (invalid/expired key).  
      - Rate limit errors if requests exceed SerpApi limits.  
      - Network timeouts or server errors.  
      - Pagination logic errors if `total_events` is not divisible by 10 or is zero.  
    - *Version Requirements:* HTTP Request node version 4.2 or later recommended for pagination features.

#### 2.3 Data Processing

- **Overview:**  
  Processes the paginated API responses, extracts and flattens nested event information, and adds the original search query to each event record for context.

- **Nodes Involved:**  
  - Process & Flatten Events  

- **Node Details:**  

  - **Process & Flatten Events**  
    - *Type:* Code (JavaScript) node  
    - *Role:* Iterates through all paginated API responses, extracts `events_results` arrays, flattens them into individual event JSON objects, and attaches the search query to each event.  
    - *Configuration:* Custom JavaScript code (ES6+), looping through input data and building an array of event objects.  
    - *Key Expressions:* Accesses properties like `item.json.events_results` and `item.json.search_parameters.q`.  
    - *Input/Output:* Input from SerpApi Events Request; outputs flattened event JSON objects for Google Sheets insertion.  
    - *Edge Cases:*  
      - Missing or empty `events_results` arrays.  
      - Unexpected data structure changes in API responses.  
      - Null or undefined query parameters.  
    - *Version Requirements:* Code node version 2+ for modern JS syntax support.

#### 2.4 Data Output

- **Overview:**  
  Appends the processed event data as rows into a Google Sheets spreadsheet, mapping specific event fields to columns for structured visualization and analysis.

- **Nodes Involved:**  
  - Save to Google Sheets  

- **Node Details:**  

  - **Save to Google Sheets**  
    - *Type:* Google Sheets node  
    - *Role:* Inserts rows into a predefined Google Sheets document and tab, appending new event entries without overwriting existing data.  
    - *Configuration:*  
      - Document ID: `1DQo3tI8yKzCbLn-DWN2hureHgwj1XxvM1ogES1_77ts` (shared spreadsheet)  
      - Sheet/GID: `gid=0` (first tab named "Sheet1")  
      - Operation: Append rows  
      - Columns mapped explicitly from event JSON fields, including title, link, description, start_date, venue, address, ticket_info, event_location_map link, image, and query.  
      - No automatic type conversion to preserve data integrity.  
      - Credential: Google Sheets OAuth2, requires user authentication.  
    - *Input/Output:* Input from Process & Flatten Events; no output (end of workflow).  
    - *Edge Cases:*  
      - OAuth2 token expiry or permission issues.  
      - Sheet access or quota limits.  
      - Data mismatches if the schema changes or data contains unexpected types.  
    - *Version Requirements:* Google Sheets node version 4.5 or higher for full features.

---

### 3. Summary Table

| Node Name               | Node Type             | Functional Role                              | Input Node(s)           | Output Node(s)           | Sticky Note                                                                                         |
|-------------------------|-----------------------|----------------------------------------------|-------------------------|--------------------------|---------------------------------------------------------------------------------------------------|
| Manual Trigger          | Manual Trigger        | Starts workflow manually                      | —                       | Set Search Parameters     |                                                                                                |
| Set Search Parameters   | Set                   | Defines search query, total events, start    | Manual Trigger          | SerpApi Events Request    | 🔧 SEARCH CONFIGURATION<br>📍 Query: "Events in Texas"<br>📊 Total Events: 20<br>🚀 Start Position: 0<br>💡 Modify these values:<br>- Change location/topic<br>- Adjust event count (10 per page)<br>- Set pagination start point<br>⚠️ Note: SerpApi rate limits apply |
| SerpApi Events Request  | HTTP Request          | Queries SerpApi Google Events API with pagination | Set Search Parameters   | Process & Flatten Events  | 🌐 SERPAPI INTEGRATION<br>🔑 Engine: google_events<br>🌍 Language: en (English)<br>📍 Country: us (United States)<br>📄 Pagination: 10 events per request, 2-second delay<br>🔐 Requires SerpApi credentials |
| Process & Flatten Events | Code                  | Flattens nested event data, adds query info | SerpApi Events Request  | Save to Google Sheets     | ⚙️ DATA TRANSFORMATION<br>📝 Flattens nested event data<br>🏷️ Adds search query to each event<br>🔄 Combines multiple API responses<br>📊 Output Fields: title, link, description, start_date, when, address, venue, ticket_info, image, event_location_map, query |
| Save to Google Sheets   | Google Sheets         | Appends processed event data to Google Sheets | Process & Flatten Events | —                        | 📋 GOOGLE SHEETS OUTPUT<br>📄 Sheet: "Google Events API"<br>📊 Tab: "Sheet1"<br>✅ Mode: Append rows<br>🔗 [Spreadsheet](https://docs.google.com/spreadsheets/d/1DQo3tI8yKzCbLn-DWN2hureHgwj1XxvM1ogES1_77ts/edit?usp=sharing)<br>🔐 Requires Google Sheets OAuth |
| Sticky Note1            | Sticky Note           | Search configuration instructions             | —                       | —                        | 🔧 SEARCH CONFIGURATION<br>📍 Query: "Events in Texas"<br>📊 Total Events: 20<br>🚀 Start Position: 0<br>💡 Modify these values:<br>- Change location/topic<br>- Adjust event count (10 per page)<br>- Set pagination start point<br>⚠️ Note: SerpApi rate limits apply |
| Sticky Note2            | Sticky Note           | SerpApi integration notes                      | —                       | —                        | 🌐 SERPAPI INTEGRATION<br>🔑 Engine: google_events<br>🌍 Language: en (English)<br>📍 Country: us (United States)<br>📄 Pagination:<br>- 10 events per request<br>- 2-second delay between calls<br>- Auto-calculates max requests<br>🔐 Requires SerpApi credentials |
| Sticky Note3            | Sticky Note           | Data transformation details                    | —                       | —                        | ⚙️ DATA TRANSFORMATION<br>📝 Flattens nested event data<br>🏷️ Adds search query to each event<br>🔄 Combines multiple API responses<br>📊 Output Fields:<br>- title, link, description<br>- start_date, when, address<br>- venue, ticket_info, image<br>- event_location_map, query |
| Sticky Note             | Sticky Note           | General workflow purpose and scalability      | —                       | —                        | 🎯 GOOGLE EVENTS SCRAPER<br>🔍 Searches Google Events via SerpApi<br>📊 Extracts event details automatically<br>💾 Saves to Google Sheets for analysis<br>⏱️ Runtime: ~30-60 seconds<br>📈 Scalable: Adjust event count as needed<br>🔄 Manual trigger for on-demand runs |
| Sticky Note4            | Sticky Note           | Google Sheets output details and link         | —                       | —                        | 📋 GOOGLE SHEETS OUTPUT<br>📄 Sheet: "Google Events API"<br>📊 Tab: "Sheet1"<br>✅ Mode: Append rows<br>🔗 [Spreadsheet](https://docs.google.com/spreadsheets/d/1DQo3tI8yKzCbLn-DWN2hureHgwj1XxvM1ogES1_77ts/edit?usp=sharing): (**Make a Copy of the Sheet**)<br>🔐 Requires Google Sheets OAuth |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a **Manual Trigger** node. No special configuration needed. This will let you start the workflow manually.

2. **Create Set Node to Define Search Parameters**  
   - Add a **Set** node named "Set Search Parameters".  
   - Set mode to "Raw" and enter JSON:  
     ```json
     {
       "query": "Events in Texas",
       "total_events": 10,
       "start": 0
     }
     ```  
   - Connect output of Manual Trigger to this node.

3. **Create HTTP Request Node for SerpApi Call**  
   - Add an **HTTP Request** node named "SerpApi Events Request".  
   - Configure:  
     - URL: `https://serpapi.com/search`  
     - Authentication: Use HTTP Query Auth credentials with your SerpApi API key.  
     - Query Parameters:  
       - `engine` = `"google_events"`  
       - `q` = `={{ $json.query }}` (from Set node)  
       - `hl` = `"en"`  
       - `gl` = `"us"`  
     - Pagination:  
       - Enable pagination with parameter `start` = `={{ $json.start }}`  
       - Maximum requests = `={{ $json.total_events / 10 }}`  
       - Request interval = 2000ms (2 seconds delay)  
     - Set "Send Query Parameters" to true.  
   - Connect output of "Set Search Parameters" to this node.

4. **Create Code Node to Process and Flatten Events**  
   - Add a **Code** node named "Process & Flatten Events".  
   - Paste the following JavaScript code:  
     ```javascript
     const allItems = $input.all();
     let flattenedEvents = [];

     for (const item of allItems) {
       const events = item.json.events_results || [];
       const query = item.json.search_parameters?.q || "";

       for (const event of events) {
         flattenedEvents.push({
           json: {
             ...event,
             query: query
           }
         });
       }
     }

     return flattenedEvents;
     ```  
   - Connect output of "SerpApi Events Request" to this node.

5. **Create Google Sheets Node to Append Data**  
   - Add a **Google Sheets** node named "Save to Google Sheets".  
   - Configure:  
     - Operation: Append rows  
     - Document ID: `1DQo3tI8yKzCbLn-DWN2hureHgwj1XxvM1ogES1_77ts` (or your copy of the sheet)  
     - Sheet Name: `gid=0` (Sheet1)  
     - Columns mapping (define below):  
       - `title` = `={{ $json.title }}`  
       - `start_date` = `={{ $json.date.start_date }}`  
       - `when` = `={{ $json.date.when }}`  
       - `address` = `={{ $json.address }}`  
       - `link` = `={{ $json.link }}`  
       - `event_location_map` = `={{ $json.event_location_map.link }}`  
       - `ticket_info` = `={{ $json.ticket_info }}`  
       - `venue` = `={{ $json.venue }}`  
       - `query` = `={{ $json.query }}`  
       - `description` = `={{ $json.description }}`  
       - `image` = `={{ $json.image }}`  
     - Ensure you have Google Sheets OAuth2 credentials configured and selected.  
   - Connect output of "Process & Flatten Events" to this node.

6. **Add Sticky Notes (Optional for Documentation)**  
   - Add sticky notes with relevant instructions, such as search configuration, API integration details, data transformation notes, and Google Sheets output information.

7. **Save and Test Workflow**  
   - Execute the manual trigger and monitor for successful data retrieval and appending to the Google Sheet.  
   - Adjust parameters such as `query` or `total_events` in the Set node as needed.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| The Google Sheets document used is shared publicly but requires making a personal copy for editing: [Google Events API Spreadsheet](https://docs.google.com/spreadsheets/d/1DQo3tI8yKzCbLn-DWN2hureHgwj1XxvM1ogES1_77ts/edit?usp=sharing) | Google Sheets Output node and user instructions |
| SerpApi usage is subject to rate limiting; respect the 2-second delay between paginated requests to avoid blocking. | SerpApi API best practices and workflow sticky notes |
| This workflow runtime is approximately 30-60 seconds depending on event count and network latency. | General workflow overview sticky note |
| Adjust `query` and `total_events` parameters to target different event topics or locations and control API usage volume. | Search configuration sticky note |

---

This structured documentation provides a complete understanding of the workflow’s purpose, design, and operational details, enabling both manual reproduction and automated processing by AI agents or developers.
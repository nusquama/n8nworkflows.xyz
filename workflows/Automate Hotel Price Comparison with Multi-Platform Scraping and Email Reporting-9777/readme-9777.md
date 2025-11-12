Automate Hotel Price Comparison with Multi-Platform Scraping and Email Reporting

https://n8nworkflows.xyz/workflows/automate-hotel-price-comparison-with-multi-platform-scraping-and-email-reporting-9777


# Automate Hotel Price Comparison with Multi-Platform Scraping and Email Reporting

### 1. Workflow Overview

This workflow automates hotel price comparison by receiving user requests with hotel search parameters expressed in natural language, scraping multiple hotel booking platforms for price data, aggregating and comparing results, and finally emailing a detailed report to the user. It is designed for scenarios where users want a quick, multi-source price comparison via a simple chat-like interface.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Parsing:** Receives incoming HTTP POST requests containing user queries, extracts and validates hotel name, city, and dates.
- **1.2 Validation & Readiness Check:** Determines if the parsed data is sufficient to proceed or if more info is needed.
- **1.3 Multi-Platform Scraping:** Sends parallel HTTP requests to scraping APIs for Booking.com, Agoda, and Expedia.
- **1.4 Aggregation & Comparison:** Consolidates the scraping results, finds the best deal, calculates average prices, and notes errors.
- **1.5 Email Report Generation:** Formats the pricing data into a rich HTML email report.
- **1.6 Email Sending & Webhook Responses:** Sends the email to the user and responds to the original webhook request with success or error messages.
- **1.7 Analytics Logging:** Saves summary data to Google Sheets for analytics and logs.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Parsing

**Overview:**  
This block captures user input from a webhook and parses the natural language query to extract core search parameters: hotel name, city, check-in and check-out dates, user email, and user name. It also handles greeting messages and validates presence of required info.

**Nodes Involved:**  
- Webhook - Receive Request  
- Parse & Validate Request

**Node Details:**  

- **Webhook - Receive Request**  
  - Type: Webhook  
  - Role: Entry point for HTTP POST requests at `/hotel-price-check`.  
  - Configuration: HTTP POST, response mode set to "responseNode" (delays response until explicitly sent).  
  - Inputs: HTTP request data.  
  - Outputs: Passes full JSON payload downstream.  
  - Edge Cases: May receive incomplete or malformed JSON; relies on client sending proper data.

- **Parse & Validate Request**  
  - Type: Code (JavaScript)  
  - Role: Parses natural language query, extracts hotel name, city, check-in/out dates, user email/name, and validates them.  
  - Configuration:  
    - Uses custom JS code with regex and logic to parse dates in multiple formats, recognize greetings, and clean hotel name strings.  
    - Handles greeting queries by returning an instructional response without proceeding further.  
    - If mandatory parameters are missing, returns an informative "missing_info" status with example input instructions.  
  - Key Variables:  
    - `query` (user input text)  
    - `hotelName`, `city`, `checkInDate`, `checkOutDate`, `userEmail`, `userName`  
  - Inputs: JSON body from webhook node.  
  - Outputs: JSON object indicating status (`greeting`, `missing_info`, or `ready`) and parsed fields.  
  - Edge Cases:  
    - Ambiguous or incomplete date formats.  
    - Missing city or hotel name.  
    - Greetings or help requests receive a friendly instructional response.  
    - Fallbacks for common cities if city not explicitly mentioned.

#### 2.2 Validation & Readiness Check

**Overview:**  
This block checks if the parsing result is ready to proceed with scraping or if the workflow should abort and notify the user.

**Nodes Involved:**  
- Check If Ready

**Node Details:**  

- **Check If Ready**  
  - Type: If node  
  - Role: Checks if the `status` field from parsing equals `"ready"`.  
  - Configuration: String comparison condition on `{{$json.status}} == "ready"`.  
  - Inputs: Output from Parse & Validate Request node.  
  - Outputs:  
    - True branch triggers scraping nodes.  
    - False branch triggers webhook response with info message (missing info or greeting).  
  - Edge Cases: Incorrect or unexpected status values cause false branch execution.

#### 2.3 Multi-Platform Scraping

**Overview:**  
This block queries three different hotel booking platforms via HTTP POST requests to an external scraping API, sending the parsed hotel, city, and dates. All requests run in parallel.

**Nodes Involved:**  
- Scrape Booking.com  
- Scrape Agoda  
- Scrape Expedia

**Node Details:**  

- **Scrape Booking.com / Agoda / Expedia** (3 nodes, similar configuration)  
  - Type: HTTP Request  
  - Role: Send POST requests to external APIs at URLs `https://api.example.com/scrape/{platform}`.  
  - Configuration:  
    - POST method, JSON body contains hotel name, city, check-in/out ISO dates, and platform identifier.  
    - Continue on failure enabled to allow partial results if one platform fails.  
  - Inputs: From Check If Ready node (only if ready).  
  - Outputs: JSON with price, currency, room type, availability, URL or error info.  
  - Edge Cases:  
    - Network errors, timeouts, or invalid responses from APIs.  
    - Failure on one platform does not block others.

#### 2.4 Aggregation & Comparison

**Overview:**  
Aggregates the price data from all platforms, sorts by price, extracts the best deal, calculates average price and potential savings, and prepares a summary object.

**Nodes Involved:**  
- Aggregate & Compare

**Node Details:**  

- **Aggregate & Compare**  
  - Type: Code (JavaScript)  
  - Role:  
    - Collects all platform results (except first item which is search data).  
    - For each platform result, checks for errors or extracts price data.  
    - Sorts prices ascending and calculates:  
      - Best deal (lowest price)  
      - Average price  
      - Savings compared to highest price  
    - Returns a status `"success"` with detailed results or `"no_results"` if no prices found.  
  - Inputs: Outputs of all three scraping HTTP requests plus initial search data.  
  - Outputs: JSON with status, results array, bestDeal object, average price, savings, errors, and original search data.  
  - Edge Cases: All platforms fail, no price data found, partial data missing.

#### 2.5 Email Report Generation

**Overview:**  
Formats an attractive HTML email report summarizing the hotel search and price comparison results, or a no-result message if no prices found.

**Nodes Involved:**  
- Format Email Report

**Node Details:**  

- **Format Email Report**  
  - Type: Code (JavaScript)  
  - Role:  
    - Checks if no results found, returns an email notifying user accordingly.  
    - Otherwise, builds an HTML email with header, hotel info, best deal highlight, price comparison table, average price, and footer.  
    - Uses inline styling for email client compatibility.  
    - Dynamically inserts all relevant data (hotel name, city, dates, prices, platforms).  
  - Inputs: Aggregated comparison results.  
  - Outputs: JSON with email subject and HTML body plus all data.  
  - Edge Cases: No results scenario handled gracefully.  
  - Notes: Email formatting emphasizes best deal with colors and icons.

#### 2.6 Email Sending & Webhook Responses

**Overview:**  
Sends the formatted email to the user’s email address and replies to the original webhook request indicating success or failure with appropriate message.

**Nodes Involved:**  
- Send Email Report  
- Webhook Response (Success)  
- Webhook Response (Info)

**Node Details:**  

- **Send Email Report**  
  - Type: Email Send  
  - Role: Sends the email report generated earlier to the user’s email.  
  - Configuration:  
    - Subject and HTML body taken from input JSON.  
    - To email extracted from user input.  
    - From email set to `noreply@hotelassistant.com`.  
    - SMTP credentials configured via `SMTP -test`.  
  - Inputs: From Format Email Report node.  
  - Outputs: On success, triggers webhook success response.  
  - Edge Cases: SMTP failures, invalid user email address.

- **Webhook Response (Success)**  
  - Type: Respond to Webhook  
  - Role: Sends JSON response to original HTTP request confirming success.  
  - Configuration: Includes hotel name, best price, platform in response.  
  - Inputs: From Send Email Report node.

- **Webhook Response (Info)**  
  - Type: Respond to Webhook  
  - Role: Sends JSON response with status and message for errors or info (e.g., missing info, greeting).  
  - Inputs: From false branch of Check If Ready and from Save to Google Sheets node (analytics fallback).

#### 2.7 Analytics Logging

**Overview:**  
Logs search and result summary data to Google Sheets for later analysis.

**Nodes Involved:**  
- Log Analytics  
- Save to Google Sheets

**Node Details:**  

- **Log Analytics**  
  - Type: Set node  
  - Role: Creates a JSON object with timestamp, query details, best price, platform, total results, and user email.  
  - Inputs: From Format Email Report node.  
  - Outputs: Passes data downstream to Google Sheets node.

- **Save to Google Sheets**  
  - Type: Google Sheets  
  - Role: Appends the analytics data as a new row in a specified Google Sheet and tab ("Analytics").  
  - Configuration: Uses service account authentication.  
  - Inputs: From Log Analytics node.  
  - Outputs: On success, triggers webhook info response as fallback.  
  - Edge Cases: Google API quota exceeded, invalid credentials.

---

### 3. Summary Table

| Node Name                | Node Type                 | Functional Role                     | Input Node(s)                | Output Node(s)                | Sticky Note                                                                                  |
|--------------------------|---------------------------|-----------------------------------|-----------------------------|------------------------------|----------------------------------------------------------------------------------------------|
| Webhook - Receive Request | Webhook                   | Receive user HTTP POST request    | -                           | Parse & Validate Request      | User sends natural language query                                                           |
| Parse & Validate Request  | Code                      | Parse query, extract & validate   | Webhook - Receive Request    | Check If Ready                | Extract hotel, city, dates                                                                  |
| Check If Ready           | If                        | Check if parsed data is ready     | Parse & Validate Request     | Scrape Booking.com, Agoda, Expedia / Webhook Response (Info) | Validate all required info                                                                  |
| Scrape Booking.com        | HTTP Request              | Scrape Booking.com prices         | Check If Ready               | Aggregate & Compare           | Compare prices, find best deal                                                              |
| Scrape Agoda              | HTTP Request              | Scrape Agoda prices               | Check If Ready               | Aggregate & Compare           | Compare prices, find best deal                                                              |
| Scrape Expedia            | HTTP Request              | Scrape Expedia prices             | Check If Ready               | Aggregate & Compare           | Compare prices, find best deal                                                              |
| Aggregate & Compare       | Code                      | Aggregate and analyze prices      | Scrape Booking.com, Agoda, Expedia | Format Email Report         | Compare prices, find best deal                                                              |
| Format Email Report       | Code                      | Format HTML email report          | Aggregate & Compare          | Send Email Report, Log Analytics | Create beautiful HTML report                                                                |
| Send Email Report         | Email Send                | Send email to user                | Format Email Report          | Webhook Response (Success)    | Send response                                                                              |
| Webhook Response (Success)| Respond to Webhook        | Respond success to HTTP request   | Send Email Report            | -                            | Send response                                                                              |
| Webhook Response (Info)   | Respond to Webhook        | Respond with info/error to client | Check If Ready (false branch), Save to Google Sheets | -                 | Send response                                                                              |
| Log Analytics             | Set                       | Prepare analytics data            | Format Email Report          | Save to Google Sheets         |                                                                                            |
| Save to Google Sheets     | Google Sheets             | Append analytics data to sheet   | Log Analytics               | Webhook Response (Info)       |                                                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Name: `Webhook - Receive Request`  
   - HTTP Method: POST  
   - Path: `hotel-price-check`  
   - Response Mode: `responseNode` (delayed response)  

2. **Create Code Node for Parsing & Validation**  
   - Name: `Parse & Validate Request`  
   - Paste the provided JavaScript code that:  
     - Extracts `hotelName`, `city`, `checkInDate`, `checkOutDate` from input JSON body `message`/`query`/`text`.  
     - Parses multiple date formats and applies defaults.  
     - Handles greeting messages and missing info with custom responses.  
   - Connect output of webhook node to this code node.

3. **Create If Node for Readiness Check**  
   - Name: `Check If Ready`  
   - Condition: String equals `{{$json.status}}` == `"ready"`  
   - Connect output of Parse & Validate Request node to this If node.

4. **Create HTTP Request Nodes for Scraping**  
   For each platform (Booking.com, Agoda, Expedia):  
   - Name: `Scrape Booking.com` / `Scrape Agoda` / `Scrape Expedia`  
   - Method: POST  
   - URL: `https://api.example.com/scrape/{platform}` (replace `{platform}` accordingly)  
   - Body Parameters (JSON):  
     - hotel: `{{$json.hotelName}}`  
     - city: `{{$json.city}}`  
     - checkIn: `{{$json.checkInISO}}`  
     - checkOut: `{{$json.checkOutISO}}`  
     - platform: `{platform}`  
   - Enable `Continue On Fail` option.  
   - Connect If node’s true branch to all three HTTP Request nodes in parallel.

5. **Create Code Node for Aggregation & Comparison**  
   - Name: `Aggregate & Compare`  
   - Paste provided JavaScript code to:  
     - Collect and parse all platform prices, errors.  
     - Sort prices, calculate best deal, average price, savings.  
     - Output status and results summary.  
   - Connect outputs of all three scraping nodes to this node.

6. **Create Code Node for Email Formatting**  
   - Name: `Format Email Report`  
   - Paste provided JavaScript code that:  
     - Builds an HTML email with hotel info, price comparison table, best deal highlight.  
     - Handles no-results scenario with an explanatory email.  
   - Connect output of Aggregation node to this node.

7. **Create Email Send Node**  
   - Name: `Send Email Report`  
   - Use SMTP credentials (configure as `SMTP -test` or your own).  
   - To: `{{$json.userEmail}}`  
   - From: `noreply@hotelassistant.com`  
   - Subject: `{{$json.subject}}`  
   - HTML Body: `{{$json.html}}`  
   - Connect output of Format Email Report to this node.

8. **Create Respond to Webhook Nodes**  
   - `Webhook Response (Success)` node:  
     - Respond JSON with success message, hotel name, best price, and platform.  
     - Connect from Email Send node.  
   - `Webhook Response (Info)` node:  
     - Respond JSON with failure or info status and message from parsing or error nodes.  
     - Connect from false branch of If node and from Google Sheets node (analytics fallback).

9. **Create Analytics Logging Nodes**  
   - `Log Analytics` (Set node):  
     - Prepare JSON with timestamp, query, hotel, city, check-in/out, best price/platform, total results, user email.  
     - Connect from Format Email Report node.  
   - `Save to Google Sheets` node:  
     - Append operation on your Google Sheets document and sheet tab (e.g., "Analytics").  
     - Use service account credentials.  
     - Connect from Log Analytics node.  
     - On success, connect to Webhook Response (Info) node for fallback response.

10. **Link False Branch of Check If Ready Node**  
    - Connect to `Webhook Response (Info)` node to return info/error messages when parsing fails or greeting detected.

11. **Final Connection Summary:**  
    - Webhook → Parse & Validate → Check If Ready  
    - Check If Ready (true) → Scrape Booking.com, Agoda, Expedia (parallel) → Aggregate & Compare → Format Email Report → Send Email Report → Webhook Response (Success)  
    - Format Email Report → Log Analytics → Save to Google Sheets → Webhook Response (Info)  
    - Check If Ready (false) → Webhook Response (Info)

---

### 5. General Notes & Resources

| Note Content                                                                                                          | Context or Link                                       |
|-----------------------------------------------------------------------------------------------------------------------|------------------------------------------------------|
| User sends natural language query like: "Hilton Hotel in Singapore from 15th March to 18th March".                     | Sticky Note on Webhook - Receive Request             |
| Extract hotel, city, dates, with fallback to common cities list.                                                      | Sticky Note on Parse & Validate Request               |
| Validation checks all required info: hotel name, city, check-in/out dates.                                             | Sticky Note on Check If Ready node                     |
| Prices compared across Booking.com, Agoda, Expedia to find best deal.                                                 | Sticky Note on scraping and aggregation nodes         |
| Creates a beautiful, mobile-friendly HTML email report with price table and best deal highlight.                       | Sticky Note on Format Email Report node                |
| Sends user email and responds to webhook with JSON success or info messages.                                          | Sticky Note on Send Email Report and Webhook Response |
| SMTP credentials must be configured for email sending, Google Sheets for analytics requires service account credentials.| Credentials setup note                                |
| External scraping APIs at `https://api.example.com/scrape/{platform}` are placeholders and must be replaced with valid scraping services or your own. | Integration detail                                     |

---

**Disclaimer:**  
The provided text is derived exclusively from an n8n automated workflow. It complies strictly with content policies and contains no illegal or offensive content. All data processed is legal and public.
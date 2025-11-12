Create Sales Orders with Web POS Interface, AI Reports, Telegram Alerts & Sheets

https://n8nworkflows.xyz/workflows/create-sales-orders-with-web-pos-interface--ai-reports--telegram-alerts---sheets-6714


# Create Sales Orders with Web POS Interface, AI Reports, Telegram Alerts & Sheets

### 1. Workflow Overview

This workflow implements a **Smart POS (Point of Sale) System** with a modern web interface for order entry, real-time integration to **Google Sheets** for product and sales data management, **AI-generated sales reports**, and **instant Telegram notifications**. It targets small businesses or vendors needing a lightweight, interactive sales platform accessible via browser.

The workflow’s logic is organized into the following blocks:

- **1.1 Web Interface & Product Data Preparation**  
  Provides a webhook serving a responsive HTML/JS web POS interface with product catalog, search, and cart functionality. It retrieves product data from Google Sheets and formats it for frontend consumption.

- **1.2 Order Submission & Processing**  
  Handles incoming POST order submissions from the web interface, validates and formats sales data, generates unique sales IDs, and appends or updates sales records into Google Sheets.

- **1.3 Order Confirmation & AI Reporting**  
  After order data is processed, it shows a confirmation page, triggers the AI agent to create a human-friendly sales report, and sends this report as a Telegram message.

---

### 2. Block-by-Block Analysis

#### 2.1 Web Interface & Product Data Preparation

- **Overview:**  
  This block exposes a webhook that serves a full-featured POS web interface in HTML/JavaScript. It reads product data from a Google Sheet, formats it into JSON strings, and injects it into the frontend for dynamic rendering and interaction.

- **Nodes Involved:**  
  - Start the webhook  
  - Get products data  
  - Format data for webhook  
  - Respond to Webhook  

- **Node Details:**

  1. **Start the webhook**  
     - Type: Webhook  
     - Role: Entry point; listens at `/smartpostsystem` path for HTTP GET requests  
     - Config: No special options; outputs to "Get products data"  
     - Edge cases: Network issues or unauthorized access could block interface access  

  2. **Get products data**  
     - Type: Google Sheets (read)  
     - Role: Reads product catalog from the first sheet (`gid=0`) in the configured Google Sheets document  
     - Config: Uses OAuth2 credential for Sheets access; executes once per webhook call  
     - Edge cases: Credential expiry, API limits, or missing sheet may cause failures  

  3. **Format data for webhook**  
     - Type: Code (JavaScript)  
     - Role: Converts product rows into separate arrays per column (`PRODUCT ID`, `PRODUCT NAME`, etc.), then stringifies them for injection into HTML/JS  
     - Key variables: `productId`, `productName`, `productImage`, `productCategoryName`, `productPriceUsd`, `productDiscount`  
     - Output: JSON object containing stringified arrays and webhook URLs for frontend  
     - Edge cases: Data inconsistencies, empty rows, or malformed data could break JSON parsing on frontend  

  4. **Respond to Webhook**  
     - Type: Respond to Webhook  
     - Role: Serves the full HTML+JS POS interface as a text/html response, embedding product data and logic for browsing, searching, and building orders  
     - Configuration: Sets `Content-Type` header as `text/html; charset=UTF-8`  
     - Edge cases: Large data payloads may slow response; HTML/JS errors impact UI  

- **Sticky Notes:**  
  - Describes this block as:  
    "- Provides webhook endpoint accessible from browser  
    - Reads and retrieves product data from Google Sheets  
    - Data is formatted and prepared for use in POS frontend or subsequent response."

---

#### 2.2 Order Submission & Processing

- **Overview:**  
  Handles order submission from the POS frontend via POST. It waits for form submission, parses and formats order details, generates unique sales IDs, and appends or updates sales data rows in Google Sheets.

- **Nodes Involved:**  
  - Wait for click  
  - Respond to click  
  - Format data for sheet  
  - Append or update row in sheet  

- **Node Details:**

  1. **Wait for click**  
     - Type: Wait (Webhook resume)  
     - Role: Suspends execution until a POST request is received on its webhook endpoint (triggered by the order form submission)  
     - Config: HTTP method POST, resumes on webhook request  
     - Edge cases: Timeout if no submission, improper POST data format  

  2. **Respond to click**  
     - Type: Respond to Webhook  
     - Role: Sends a simple confirmation HTML page to the browser after order submission, showing success message and auto-redirecting after 3 seconds  
     - Config: Inline HTML with embedded CSS and JS for redirection to next step webhook  
     - Edge cases: Browser incompatibility, network latency  

  3. **Format data for sheet**  
     - Type: Code (JavaScript)  
     - Role: Parses POST data body, extracts order items and customer name, generates a unique sales ID, and creates data rows formatted for Google Sheets insertion  
     - Key logic:  
       - Generates sales ID as `S-<timestamp>-<random4digits>`  
       - Calculates discount per item as difference between original and final price  
       - Sets sales date as today’s date (YYYY-MM-DD)  
     - Output: Array of sales data objects with keys like `SALES ID`, `SALES DATE`, `SALES CUSTOMER NAME`, `SALES PRODUCT NAME`, etc.  
     - Edge cases: Malformed JSON in order items, missing fields  

  4. **Append or update row in sheet**  
     - Type: Google Sheets (append/update)  
     - Role: Adds or updates sales data rows into the configured Google Sheet sales tab  
     - Config: Matches on `SALES ID` column to append or update; uses OAuth2 credentials  
     - Edge cases: API limits, concurrency issues, credential expiry  

- **Sticky Notes:**  
  - Describes key steps:  
    "- Creates sales data in format suitable for Google Sheets.  
    - Saves formatted sales results to sales sheet in Google Sheets file."

---

#### 2.3 Order Confirmation & AI Reporting

- **Overview:**  
  After processing the order, this block generates a simple sales report using an AI language model, formats that report into a friendly message, and sends it as a Telegram notification.

- **Nodes Involved:**  
  - Respond to click (output to) AI Agent and Format data for sheet  
  - AI Agent  
  - OpenRouter Chat Model (AI language model node)  
  - Send a text message (Telegram)  

- **Node Details:**

  1. **AI Agent**  
     - Type: LangChain Agent (AI text generation)  
     - Role: Receives order JSON data (customer name, order items, totals) and generates a human-friendly sales report message  
     - Configuration:  
       - System message instructs the model to generate a simple, friendly sales report with emojis, formatted numbers to 2 decimals, no markdown or special characters that break formatting  
       - Prompt includes placeholders for customer name and order details from the webhook data  
     - Edge cases: API throttling, unexpected input format, or AI model errors  

  2. **OpenRouter Chat Model**  
     - Type: LangChain Chat Model (OpenRouter GPT)  
     - Role: Executes the AI text generation model using the OpenRouter API  
     - Configuration: Uses the `google/gemini-2.0-flash-exp:free` model with provided OpenRouter API credentials  
     - Edge cases: API key invalid, network issues, model unavailability  

  3. **Send a text message**  
     - Type: Telegram node  
     - Role: Sends the AI-generated sales report text to a configured Telegram chat ID  
     - Configuration: Uses Telegram Bot credentials (Bot Token), sends message without attribution, message text pulled from AI Agent output  
     - Edge cases: Telegram API limits, invalid chat ID, bot not authorized  

- **Sticky Notes:**  
  - Describes functionality:  
    "- Receives order data, converts it to business owner-friendly sales report (using LLM), then sends the report to Telegram."

---

### 3. Summary Table

| Node Name               | Node Type                     | Functional Role                                   | Input Node(s)          | Output Node(s)                        | Sticky Note                                                                                   |
|-------------------------|-------------------------------|-------------------------------------------------|-----------------------|-------------------------------------|----------------------------------------------------------------------------------------------|
| Start the webhook       | Webhook                       | Entry point, serves POS interface                |                       | Get products data                   | - Provides webhook endpoint accessible from browser<br>- Reads and retrieves product data from Google Sheets<br>- Data is formatted and prepared for use in POS frontend or subsequent response. |
| Get products data       | Google Sheets                 | Reads product catalog from Google Sheets         | Start the webhook      | Format data for webhook             | Same as above                                                                               |
| Format data for webhook | Code                         | Formats product data into JSON strings for frontend | Get products data      | Respond to Webhook                  | Same as above                                                                               |
| Respond to Webhook      | Respond to Webhook            | Serves the POS HTML/JS frontend                   | Format data for webhook | Wait for click                     | - Creates POS interface in HTML format<br>- Receives order data from HTML form submitted by user using POST method |
| Wait for click          | Wait (Webhook resume)         | Waits for order submission POST                   | Respond to Webhook     | Respond to click                   | - Creates POS interface in HTML format<br>- Receives order data from HTML form submitted by user using POST method |
| Respond to click        | Respond to Webhook            | Shows order confirmation page                      | Wait for click         | AI Agent, Format data for sheet    | - Receives order data, converts it to business owner-friendly sales report (using LLM), then sends the report to Telegram. |
| Format data for sheet   | Code                         | Formats incoming order data for Google Sheets     | Respond to click       | Append or update row in sheet       | - Creates sales data in format suitable for Google Sheets.<br>- Saves formatted sales results to sales sheet in Google Sheets file. |
| Append or update row in sheet | Google Sheets          | Saves order sales data to Google Sheets           | Format data for sheet  |                                     | Same as above                                                                               |
| AI Agent                | LangChain Agent              | Generates AI sales report text                     | Respond to click       | Send a text message                | - Receives order data, converts it to business owner-friendly sales report (using LLM), then sends the report to Telegram. |
| OpenRouter Chat Model   | LangChain Chat Model         | Executes AI language model on OpenRouter           | AI Agent (ai_languageModel) | AI Agent                        | Same as above                                                                               |
| Send a text message     | Telegram                     | Sends AI-generated sales report to Telegram chat | AI Agent               |                                     | Same as above                                                                               |
| Sticky Note             | Sticky Note                  | Documentation and notes                            |                       |                                     | See detailed notes below                                                                    |
| Sticky Note1            | Sticky Note                  | Documentation                                      |                       |                                     | See detailed notes below                                                                    |
| Sticky Note2            | Sticky Note                  | Documentation                                      |                       |                                     | See detailed notes below                                                                    |
| Sticky Note3            | Sticky Note                  | Documentation                                      |                       |                                     | See detailed notes below                                                                    |
| Sticky Note4            | Sticky Note                  | Documentation                                      |                       |                                     | See detailed notes below                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook node "Start the webhook"**  
   - Type: Webhook  
   - Path: `smartpostsystem`  
   - Response Mode: `responseNode`  
   - No credentials needed  
   - Connect output to "Get products data"  

2. **Create Google Sheets node "Get products data"**  
   - Operation: Read rows  
   - Sheet Name: First sheet (`gid=0`) of your Google Sheets document  
   - Document ID: Your Google Sheets ID (replace placeholder)  
   - Credentials: Google Sheets OAuth2 account  
   - Execute once per webhook call  
   - Connect output to "Format data for webhook"  

3. **Create Code node "Format data for webhook"**  
   - Language: JavaScript  
   - Paste code to extract columns into arrays and JSON stringify them:  
     ```javascript
     const input = $input.all();
     const productId = input.map(item => item.json["PRODUCT ID"]);
     const productName = input.map(item => item.json["PRODUCT NAME"]);
     const productImage = input.map(item => item.json["PRODUCT IMAGE"]);
     const productCategoryName = input.map(item => item.json["PRODUCT CATEGORY NAME"]);
     const productPriceUsd = input.map(item => item.json["PRODUCT PRICE (USD)"]);
     const productDiscount = input.map(item => item.json["PRODUCT DISCOUNT"]);

     return [{
       json: {
         productId: JSON.stringify(productId),
         productName: JSON.stringify(productName),
         productImage: JSON.stringify(productImage),
         productCategoryName: JSON.stringify(productCategoryName),
         productPriceUsd: JSON.stringify(productPriceUsd),
         productDiscount: JSON.stringify(productDiscount),
         webhookUrl: '{{ $json.webhookUrl }}',
         resumeWebhookUrl: '{{ $resumeWebhookUrl }}'
       }
     }];
     ```  
   - Connect output to "Respond to Webhook"  

4. **Create Respond to Webhook node "Respond to Webhook"**  
   - Response with: Text  
   - Response Body: Paste the complete HTML+JS code for the POS interface provided  
   - Add HTTP header: `Content-Type: text/html; charset=UTF-8`  
   - Connect output to "Wait for click"  

5. **Create Wait node "Wait for click"**  
   - Wait for webhook resume on POST method  
   - Resume on webhook request  
   - Connect output to "Respond to click"  

6. **Create Respond to Webhook node "Respond to click"**  
   - Response with: Text  
   - Response Body: Use the HTML confirmation page with auto-redirect after 3 seconds  
   - Connect output to two nodes: "AI Agent" and "Format data for sheet"  

7. **Create Code node "Format data for sheet"**  
   - Language: JavaScript  
   - Paste code to parse order data, generate sales ID and format rows:  
     ```javascript
     const data = $input.first().json.body;
     const items = JSON.parse(data.orderItems);

     function generateSalesId() {
       const timestamp = Date.now();
       const random = Math.floor(Math.random() * 10000).toString().padStart(4, '0');
       return `S-${timestamp}-${random}`;
     }

     const today = new Date().toISOString().split('T')[0];
     const salesId = generateSalesId();

     const output = items.map(item => {
       const discount = Number((item.originalPrice - item.price).toFixed(2));
       return {
         'SALES ID': salesId,
         'SALES DATE': today,
         'SALES CUSTOMER NAME': data.customerName,
         'SALES PRODUCT NAME': item.name,
         'SALES CATEGORY NAME': '',
         'SALES PRICE (USD)': item.price,
         'SALES QTY': item.quantity,
         'SALES DISCOUNT': discount,
         'SALES TOTAL': item.total
       };
     });

     return output.map(item => ({ json: item }));
     ```  
   - Connect output to "Append or update row in sheet"  

8. **Create Google Sheets node "Append or update row in sheet"**  
   - Operation: Append or update  
   - Document ID: Your Google Sheets ID  
   - Sheet Name: Your sales sheet tab ID  
   - Matching Columns: `SALES ID`  
   - Map columns for sales ID, date, customer name, product name, price, qty, discount, total  
   - Credentials: Google Sheets OAuth2  
   - Connect output to end (no further nodes)  

9. **Create LangChain Agent node "AI Agent"**  
   - Input text:  
     ```
     customer name : {{ $json.body.customerName }}
     order items : {{ $json.body.orderItems }}
     order total : {{ $json.body.orderTotals }}

     Sales report format : 
     New sales! (opening or greetings to the owner )
     customer name : 
     order details :
     Have a good day (closing)
     ```  
   - System message:  
     ```
     You are a virtual assistant whose primary task is to create sales reports for business owners.
     Write in a simple and friendly format. Use emojis to make it more interactive.
     Some item prices are separated by commas.
     Format all numbers such as prices, subtotal, and total to 2 decimal places only (e.g., 12.97, not 12.969999999999999).
     Avoid using long floating-point numbers.

     Avoid using special characters that may break Markdown formatting, such as:
     *, _, [, ], (, ), ~, >, #, +, -, =, {, }, ., !, $.
     Use plain text without special symbols unless necessary.
     Do not use Markdown or HTML formatting.
     ```  
   - Connect AI language model parameter to "OpenRouter Chat Model"  

10. **Create LangChain Chat Model node "OpenRouter Chat Model"**  
    - Model: `google/gemini-2.0-flash-exp:free`  
    - Credentials: OpenRouter API key  
    - Connect output back to "AI Agent" node  

11. **Create Telegram node "Send a text message"**  
    - Chat ID: Your Telegram chat ID  
    - Text: `={{ $json.output }}` (output text from AI Agent)  
    - Credential: Telegram Bot API token  
    - Connect input from "AI Agent" node  

12. **Add Sticky Notes** for documentation and guidance as per original workflow content.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                        | Context or Link                                                                                     |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Smart POS System with Live Updates to Telegram & Sheets. This template provides a lightweight sales management solution integrating a modern web interface, Google Sheets, Telegram notifications, and AI-generated reports. Ideal for small businesses or mobile vendors.                                                                                                                         | Full project description included in Sticky Note at workflow start                                 |
| Key requirements: Google Sheets template (link to be replaced with your actual sheet ID), Telegram Bot Token and Chat ID (created via @BotFather), OpenRouter API Key (sign up at https://openrouter.ai).                                                                                                                                                                                             | Telegram Bot creation guide: https://t.me/BotFather; OpenRouter signup: https://openrouter.ai      |
| The POS interface uses Bootstrap 4 and FontAwesome for styling and icons, providing responsive UI with product filtering, search, categories, cart management with quantity and discount handling.                                                                                                                                                                                                 | See embedded HTML+JS in "Respond to Webhook" node                                                 |
| Sales IDs are generated uniquely using timestamp and random digits to avoid collisions. Discounts are calculated per item and reported accurately. Numbers are formatted to two decimal places to ensure currency consistency.                                                                                                                                                                        | See code in "Format data for sheet" node                                                         |
| AI sales report uses a friendly tone with emojis, avoids special characters that break formatting, and formats numbers properly. The Telegram notification is immediate after order processing, ensuring owner is alerted in real-time.                                                                                                                                                                | See "AI Agent" node prompt and system message                                                    |
| Google Sheets integration uses OAuth2 credentials; ensure credentials are active and have required scopes. API limits for Sheets and Telegram apply; handle errors accordingly.                                                                                                                                                                                                                      | Credential setup reminders                                                                        |

---

This structured and detailed reference enables developers and AI agents to fully understand, reproduce, and modify the Smart POS System workflow, while anticipating potential integration issues and edge cases.
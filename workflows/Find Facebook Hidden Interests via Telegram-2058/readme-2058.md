Find Facebook Hidden Interests via Telegram

https://n8nworkflows.xyz/workflows/find-facebook-hidden-interests-via-telegram-2058


# Find Facebook Hidden Interests via Telegram

### 1. Workflow Overview

This n8n workflow automates the extraction and analysis of user-specified interests from Telegram messages, leveraging the Facebook Graph API to find detailed information about those interests. The workflow is targeted at users who want to monitor or analyze interests shared in a Telegram chat, streamlining data retrieval and presentation in a structured format.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Filtering:** Listens to Telegram messages and filters for relevant messages starting with "#interest" from a specific chat.
- **1.2 Interest Extraction and Processing:** Extracts the interest term from the message and prepares it for querying.
- **1.3 Facebook Graph API Query:** Searches Facebook’s ad interest data using the extracted interest.
- **1.4 Data Transformation:** Transforms and organizes the API response data into a structured table of relevant fields.
- **1.5 Output Generation and Delivery:** Creates a CSV spreadsheet from the data and sends it back to the Telegram chat.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Filtering

**Overview:**  
This block listens for incoming Telegram messages, checks if the message is from a specific chat, and filters messages starting with the hashtag "#interest" to determine if processing should continue.

**Nodes Involved:**  
- Get interest name (Telegram Trigger)  
- Check message contents (If)

**Node Details:**

- **Get interest name**  
  - Type: Telegram Trigger  
  - Role: Entry point that listens for new messages via Telegram webhook.  
  - Configuration: Triggers on "message" updates with Telegram API credentials.  
  - Key variables: Incoming message JSON with chat ID and text.  
  - Output: Emits message JSON to downstream nodes.  
  - Edge cases:  
    - Missing or invalid Telegram credentials may cause trigger failure.  
    - Telegram webhook misconfiguration could prevent message reception.  
  - No sub-workflows.

- **Check message contents**  
  - Type: If  
  - Role: Filters incoming messages to pass only those from a specific Telegram chat (chat ID `-1001805495093`) and starting with "#interest".  
  - Configuration:  
    - Numeric condition: `$json.message.chat.id == -1001805495093`  
    - String condition: `$json.message.text` starts with "#interest"  
  - Inputs: From Telegram Trigger node.  
  - Outputs:  
    - True branch proceeds with extraction.  
    - False branch triggers a No Operation node to ignore irrelevant messages.  
  - Edge cases:  
    - Messages without `text` property or malformed JSON will fail condition evaluation.  
    - Chat ID or hashtag mismatch results in messages being ignored.  
  - No sub-workflows.

- **No Operation, do nothing**  
  - Type: NoOp  
  - Role: Terminates workflow cleanly for messages that do not match criteria.  
  - Configuration: No parameters.  
  - Input: False branch of Check message contents node.  
  - Output: None.  
  - Edge cases: None.

---

#### 2.2 Interest Extraction and Processing

**Overview:**  
Extracts the textual content of the Telegram message and parses the interest hashtag and remaining text for querying.

**Nodes Involved:**  
- Extract message (Code)  
- Split Message (Code)

**Node Details:**

- **Extract message**  
  - Type: Code (JavaScript)  
  - Role: Extracts the raw text from the incoming Telegram message JSON.  
  - Configuration: Custom JS code that reads `message.text` safely and outputs `messageContent`.  
  - Key expressions:  
    ```js
    let messageContent = inputData.message?.text || '';
    ```
  - Input: Message JSON from Check message contents node (True branch).  
  - Output: JSON with `messageContent` string.  
  - Edge cases:  
    - Missing or empty `text` property results in empty `messageContent`.  
    - Input JSON format changes could break extraction.  
  - No sub-workflows.

- **Split Message**  
  - Type: Code (JavaScript)  
  - Role: Parses the `messageContent` to extract the interest keyword following the hashtag and any remaining content.  
  - Configuration: Uses regex `/#(\w+)\b(.*)/` to separate first hashtag word and remainder.  
  - Key expressions:  
    ```js
    let regex = /#(\w+)\b(.*)/;
    let matches = regex.exec(variableContent);
    let extractedContent = matches ? matches[1] : '';
    let remainingContent = matches ? matches[2].trim() : variableContent.trim();
    ```
  - Input: Output from Extract message node.  
  - Output: JSON with `extractedContent` (interest term) and `remainingContent`.  
  - Edge cases:  
    - Messages without a proper hashtag format produce empty `extractedContent` and full message as remaining content.  
    - Unexpected message formatting may cause incorrect extraction.  
  - No sub-workflows.

---

#### 2.3 Facebook Graph API Query

**Overview:**  
Queries the Facebook Graph API's ad interest search endpoint using the extracted interest to retrieve detailed data on matching ad interests.

**Nodes Involved:**  
- Connect to Graph API (Facebook Graph API)

**Node Details:**

- **Connect to Graph API**  
  - Type: Facebook Graph API  
  - Role: Searches Facebook ad interests with the remaining content as the query string.  
  - Configuration:  
    - Node endpoint: `search?type=adinterest&q={{ $json.remainingContent }}&limit=1000000&locale=en_US`  
    - API version: v17.0  
    - Uses Facebook Graph API credentials.  
  - Inputs: From Split Message node.  
  - Outputs: JSON response containing ad interest data.  
  - Edge cases:  
    - API authentication failure due to invalid or expired token.  
    - API rate limits or quota exceeded.  
    - Large data sets may cause timeout or performance issues.  
    - Malformed queries if `remainingContent` is empty or invalid.  
  - No sub-workflows.

- **Sticky Note (related to this block)**  
  - Content:  
    ```
    ## Facebook API

    To get the API Key you need to follow these steps:
    https://developers.facebook.com/docs/commerce-platform/setup/api-setup/
    ```  
  - Positioned near Connect to Graph API node as a reminder for credential setup.

---

#### 2.4 Data Transformation

**Overview:**  
Transforms the nested JSON response from the Facebook API into a flat table format, then extracts key variables for report generation.

**Nodes Involved:**  
- Split Interests into a Table (Code)  
- Get variables (Code)

**Node Details:**

- **Split Interests into a Table**  
  - Type: Code (JavaScript)  
  - Role: Iterates over the JSON response object and converts nested key-value pairs into a flat list of `{Item, SubItem, Value}` objects.  
  - Configuration: Custom JS looping over each key and subkey in input JSON.  
  - Input: Facebook Graph API JSON response.  
  - Output: Array of JSON objects with keys `Item`, `SubItem`, and `Value` (where `Value` is itself an object containing interest details).  
  - Edge cases:  
    - Unexpected data structure can cause errors or empty output.  
    - Large datasets could impact performance.  
  - No sub-workflows.

- **Get variables**  
  - Type: Code (JavaScript)  
  - Role: Maps the flattened table to extract specific fields (`name`, audience size bounds, `path`, `description`, `topic`) from each interest for the spreadsheet.  
  - Configuration:  
    ```js
    items.map(item => ({
      json: {
        name: item.json.Value.name,
        audience_size_lower_bound: item.json.Value.audience_size_lower_bound,
        audience_size_upper_bound: item.json.Value.audience_size_upper_bound,
        path: item.json.Value.path,
        description: item.json.Value.description,
        topic: item.json.Value.topic,
      }
    }));
    ```  
  - Input: Output from Split Interests into a Table.  
  - Output: JSON array formatted for spreadsheet generation.  
  - Edge cases:  
    - Missing fields in `Value` objects cause undefined entries.  
    - Improper input data format can cause runtime errors.  
  - No sub-workflows.

---

#### 2.5 Output Generation and Delivery

**Overview:**  
Generates a CSV spreadsheet from the processed interest data and sends this file back to the originating Telegram chat as a document attachment.

**Nodes Involved:**  
- Create a Spreadsheet (Spreadsheet File)  
- Send the Spreadsheet file (Telegram)

**Node Details:**

- **Create a Spreadsheet**  
  - Type: Spreadsheet File  
  - Role: Converts the JSON data into a CSV file stored as binary data within the workflow.  
  - Configuration:  
    - Operation: `toFile`  
    - File format: `csv`  
    - No additional options specified.  
  - Input: JSON array from Get variables node.  
  - Output: Binary file data representing `report.csv`.  
  - Edge cases:  
    - Large data may cause memory or processing delays.  
    - Malformed input JSON may cause conversion errors.  
  - No sub-workflows.

- **Send the Spreadsheet file**  
  - Type: Telegram  
  - Role: Sends the generated CSV file to the Telegram chat specified by chat ID.  
  - Configuration:  
    - Chat ID: `-1001805495093` (same as source)  
    - Operation: `sendDocument`  
    - Binary data enabled, file named `report.csv`  
    - Uses Telegram API credentials (same as trigger).  
  - Input: Binary file from Create a Spreadsheet node.  
  - Output: Telegram message with document sent.  
  - Edge cases:  
    - Telegram API errors due to invalid token or network issues.  
    - File size limits exceeded on Telegram side.  
  - No sub-workflows.

---

### 3. Summary Table

| Node Name                | Node Type              | Functional Role                                | Input Node(s)             | Output Node(s)              | Sticky Note                                                                                  |
|--------------------------|------------------------|-----------------------------------------------|---------------------------|-----------------------------|----------------------------------------------------------------------------------------------|
| Get interest name        | Telegram Trigger       | Receives Telegram messages                     | —                         | Check message contents       |                                                                                              |
| Check message contents   | If                     | Filters messages by chat ID and hashtag prefix| Get interest name          | Extract message, No Operation|                                                                                              |
| No Operation, do nothing | NoOp                   | Terminates workflow for irrelevant messages   | Check message contents     | —                           |                                                                                              |
| Extract message          | Code                   | Extracts raw text from Telegram message        | Check message contents     | Split Message               |                                                                                              |
| Split Message            | Code                   | Parses hashtag and remaining content           | Extract message            | Connect to Graph API        |                                                                                              |
| Connect to Graph API     | Facebook Graph API     | Queries Facebook ad interests API              | Split Message              | Split Interests into a Table| ## Facebook API<br>To get the API Key you need to follow these steps:<br>https://developers.facebook.com/docs/commerce-platform/setup/api-setup/ |
| Split Interests into a Table | Code               | Flattens nested API response into table format | Connect to Graph API       | Get variables               |                                                                                              |
| Get variables            | Code                   | Extracts relevant fields for spreadsheet       | Split Interests into a Table| Create a Spreadsheet        |                                                                                              |
| Create a Spreadsheet     | Spreadsheet File       | Generates CSV file from JSON data               | Get variables              | Send the Spreadsheet file   |                                                                                              |
| Send the Spreadsheet file| Telegram               | Sends CSV file back to Telegram chat            | Create a Spreadsheet       | —                           |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node:**  
   - Type: Telegram Trigger  
   - Set "Updates" to `message`.  
   - Configure credentials with a Telegram Bot API token.  
   - Define webhook ID or let n8n generate one.  
   - Position accordingly.

2. **Add If Node (Check message contents):**  
   - Type: If  
   - Add Numeric Condition:  
     - Value1: `{{$json["message"]["chat"]["id"]}}`  
     - Condition: Equal  
     - Value2: `-1001805495093` (Telegram chat ID)  
   - Add String Condition:  
     - Value1: `{{$json["message"]["text"]}}`  
     - Condition: Starts With  
     - Value2: `#interest`  
   - Connect Telegram Trigger output to If node input.

3. **Add No Operation Node:**  
   - Type: NoOp  
   - Connect If node False output to NoOp node.

4. **Add Code Node (Extract message):**  
   - Type: Code (JavaScript)  
   - Code:  
     ```js
     let inputData = items[0].json;
     let messageContent = inputData.message?.text || '';
     return [{ json: { messageContent } }];
     ```  
   - Connect If node True output to this node.

5. **Add Code Node (Split Message):**  
   - Type: Code (JavaScript)  
   - Code:  
     ```js
     let variableContent = String(items[0].json.messageContent || '');
     let regex = /#(\w+)\b(.*)/;
     let matches = regex.exec(variableContent);
     let extractedContent = matches ? matches[1] : '';
     let remainingContent = matches ? matches[2].trim() : variableContent.trim();
     return [{ json: { extractedContent, remainingContent } }];
     ```  
   - Connect Extract message node output to this node.

6. **Add Facebook Graph API Node:**  
   - Type: Facebook Graph API  
   - Credentials: Set up Facebook Graph API credentials with valid access token.  
   - Operation: Custom Request  
   - HTTP Method: GET  
   - Resource URL: `search`  
   - Query Parameters:  
     - `type`: `adinterest`  
     - `q`: `{{$json["remainingContent"]}}`  
     - `limit`: `1000000`  
     - `locale`: `en_US`  
   - API Version: `v17.0`  
   - Connect Split Message node output to this node.

7. **Add Code Node (Split Interests into a Table):**  
   - Type: Code (JavaScript)  
   - Code:  
     ```js
     let inputData = items[0].json;
     let outputData = [];
     for (let key in inputData) {
       if (inputData.hasOwnProperty(key)) {
         let itemValue = inputData[key];
         for (let subKey in itemValue) {
           if (itemValue.hasOwnProperty(subKey)) {
             outputData.push({
               json: {
                 Item: key,
                 SubItem: subKey,
                 Value: itemValue[subKey]
               }
             });
           }
         }
       }
     }
     return outputData;
     ```  
   - Connect Facebook Graph API node output to this node.

8. **Add Code Node (Get variables):**  
   - Type: Code (JavaScript)  
   - Code:  
     ```js
     return items.map(item => {
       let data = item.json.Value;
       return {
         json: {
           name: data.name,
           audience_size_lower_bound: data.audience_size_lower_bound,
           audience_size_upper_bound: data.audience_size_upper_bound,
           path: data.path,
           description: data.description,
           topic: data.topic
         }
       };
     });
     ```  
   - Connect Split Interests into a Table node output to this node.

9. **Add Spreadsheet File Node (Create a Spreadsheet):**  
   - Type: Spreadsheet File  
   - Operation: `toFile`  
   - File Format: `csv`  
   - Connect Get variables node output to this node.

10. **Add Telegram Node (Send the Spreadsheet file):**  
    - Type: Telegram  
    - Operation: `sendDocument`  
    - Chat ID: `-1001805495093` (same chat as source)  
    - Enable Binary Data and set file name as `report.csv`  
    - Credentials: Use same Telegram API credentials as trigger node.  
    - Connect Create a Spreadsheet node output to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                                         |
|-------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| To get the Facebook API Key, follow the official Facebook Commerce Platform API setup guide.     | https://developers.facebook.com/docs/commerce-platform/setup/api-setup/                                                |
| The workflow assumes the Telegram chat ID is `-1001805495093`. Modify accordingly if needed.     | Chat ID used for filtering and sending messages.                                                                        |
| The regex for extracting hashtags expects a single hashtag followed by optional text.            | Modify regex in Split Message node if message format varies.                                                            |
| Facebook Graph API rate limits and API version changes may affect workflow stability over time.  | Monitor API announcements and update node configurations as necessary.                                                  |
| Use n8n credentials manager to securely store Telegram and Facebook API tokens.                   | Improves security and ease of maintenance.                                                                               |

---

This document provides a complete, detailed reference to understand, reproduce, and maintain the "Find Facebook Hidden Interests via Telegram" n8n workflow.
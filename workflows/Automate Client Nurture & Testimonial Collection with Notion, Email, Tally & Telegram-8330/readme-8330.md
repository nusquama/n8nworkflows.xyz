Automate Client Nurture & Testimonial Collection with Notion, Email, Tally & Telegram

https://n8nworkflows.xyz/workflows/automate-client-nurture---testimonial-collection-with-notion--email--tally---telegram-8330


# Automate Client Nurture & Testimonial Collection with Notion, Email, Tally & Telegram

### 1. Workflow Overview

This workflow automates client nurture email sequences based on data retrieved from Notion, targeting specific client engagement days (7, 30, and 60 days after a reference date). It is designed for businesses that manage client information in Notion and want to send timely, personalized email follow-ups without manual intervention. The logical flow consists of:

- **1.1 Input Reception and Initialization:** Manual trigger initiation and current date retrieval.
- **1.2 Notion Data Retrieval:** Fetch client data from Notion.
- **1.3 Calculation & Decision Logic:** Compute elapsed days since a key date and branch logic based on day intervals (7, 30, 60).
- **1.4 Email Dispatch:** Send specific nurture emails corresponding to each milestone day.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Initialization

- **Overview:**  
  This block starts the workflow manually and retrieves the current date and time to serve as a reference point for subsequent calculations.

- **Nodes Involved:**  
  - Manual Trigger  
  - Get Date

- **Node Details:**  

  - **Manual Trigger**  
    - Type: Trigger node  
    - Role: Initiates the workflow manually when an operator triggers it.  
    - Configuration: No parameters needed; purely manual start.  
    - Inputs: None  
    - Outputs: Triggers the "Get Date" node.  
    - Edge cases: No failures expected unless manual trigger is never activated.

  - **Get Date**  
    - Type: Code node (JavaScript)  
    - Role: Obtains the current date and time, standardizing the reference date for calculations.  
    - Configuration: Executes a JavaScript snippet to get the current timestamp or formatted date.  
    - Inputs: Receives trigger from Manual Trigger.  
    - Outputs: Passes the current date to the Notion node.  
    - Edge cases: Code errors or timezone inconsistencies can cause incorrect date retrieval.

---

#### 2.2 Notion Data Retrieval

- **Overview:**  
  Queries the Notion database or page(s) to extract client-related data needed to determine email timing.

- **Nodes Involved:**  
  - Notion

- **Node Details:**  

  - **Notion**  
    - Type: Notion integration node  
    - Role: Retrieves client records from Notion workspace based on configured query or database ID.  
    - Configuration: Connected with valid Notion credentials; queries database or page(s) with filters as appropriate.  
    - Inputs: Receives current date context from "Get Date".  
    - Outputs: Sends client data to "Calculate Days" node.  
    - Version: Version 2.2 ensures compatibility with latest Notion API changes.  
    - Edge cases: Authentication failures, API rate limits, empty or malformed data can disrupt flow.

---

#### 2.3 Calculation & Decision Logic

- **Overview:**  
  Computes the number of days elapsed since a client-specific date and routes the workflow based on whether it matches 7, 30, or 60 days.

- **Nodes Involved:**  
  - Calculate Days  
  - Day 7  
  - Day 30  
  - Day 60

- **Node Details:**  

  - **Calculate Days**  
    - Type: Code node (JavaScript)  
    - Role: Uses client date from Notion and current date to calculate days elapsed per client.  
    - Configuration: Custom code calculates difference in days, outputs result for branching.  
    - Inputs: Receives client data and current date.  
    - Outputs: Provides results to three conditional "If" nodes.  
    - Edge cases: Date format mismatches, missing data, or invalid date fields can cause calculation errors.

  - **Day 7**  
    - Type: If node (conditional branching)  
    - Role: Checks if days elapsed equals 7.  
    - Configuration: Condition compares calculated days to 7 (exact match or range).  
    - Inputs: Receives calculated days.  
    - Outputs: On true, triggers "Email 7". On false, passes control to other conditionals.  
    - Edge cases: Off-by-one errors if date calculations are inaccurate.

  - **Day 30**  
    - Type: If node  
    - Role: Checks if days elapsed equals 30.  
    - Configuration: Similar to Day 7 node but for day 30.  
    - Inputs: Receives calculated days.  
    - Outputs: On true, triggers "Email 30".  
    - Edge cases: Same as Day 7 node.

  - **Day 60**  
    - Type: If node  
    - Role: Checks if days elapsed equals 60.  
    - Configuration: Similar to Day 7 node but for day 60.  
    - Inputs: Receives calculated days.  
    - Outputs: On true, triggers "Email 60".  
    - Edge cases: Same as previous conditionals.

---

#### 2.4 Email Dispatch

- **Overview:**  
  Sends automated nurture emails to clients based on the elapsed time milestone reached.

- **Nodes Involved:**  
  - Email 7  
  - Email 30  
  - Email 60

- **Node Details:**  

  - **Email 7**  
    - Type: Email Send node  
    - Role: Sends the day 7 nurture email.  
    - Configuration: SMTP or other email credentials configured; email body and recipients set dynamically from Notion data.  
    - Inputs: Triggered from "Day 7" node.  
    - Outputs: Ends workflow branch after sending email.  
    - Edge cases: SMTP authentication failures, email formatting issues, invalid recipient addresses.

  - **Email 30**  
    - Type: Email Send node  
    - Role: Sends the day 30 nurture email.  
    - Configuration: Similar to "Email 7" node with different content.  
    - Inputs: Triggered from "Day 30" node.  
    - Outputs: Ends workflow branch after sending email.  
    - Edge cases: Same as "Email 7".

  - **Email 60**  
    - Type: Email Send node  
    - Role: Sends the day 60 nurture email.  
    - Configuration: Similar to previous email nodes with appropriate message content.  
    - Inputs: Triggered from "Day 60" node.  
    - Outputs: Ends workflow branch after sending email.  
    - Edge cases: Same as previous email nodes.

---

### 3. Summary Table

| Node Name    | Node Type           | Functional Role                        | Input Node(s) | Output Node(s) | Sticky Note |
|--------------|---------------------|-------------------------------------|---------------|----------------|-------------|
| Manual Trigger | Manual Trigger      | Initiates the workflow manually     | None          | Get Date       |             |
| Get Date     | Code                | Retrieves current date/time          | Manual Trigger| Notion         |             |
| Notion       | Notion              | Fetches client data from Notion      | Get Date      | Calculate Days |             |
| Calculate Days | Code              | Calculates elapsed days since client date | Notion    | Day 7, Day 30, Day 60 |             |
| Day 7        | If                  | Checks if 7 days elapsed              | Calculate Days| Email 7        |             |
| Day 30       | If                  | Checks if 30 days elapsed             | Calculate Days| Email 30       |             |
| Day 60       | If                  | Checks if 60 days elapsed             | Calculate Days| Email 60       |             |
| Email 7      | Email Send          | Sends nurture email for day 7         | Day 7         | None           |             |
| Email 30     | Email Send          | Sends nurture email for day 30        | Day 30        | None           |             |
| Email 60     | Email Send          | Sends nurture email for day 60        | Day 60        | None           |             |
| Sticky Note  | Sticky Note         | (No content)                         | None          | None           |             |
| Sticky Note1 | Sticky Note         | (No content)                         | None          | None           |             |
| Sticky Note2 | Sticky Note         | (No content)                         | None          | None           |             |
| Sticky Note3 | Sticky Note         | (No content)                         | None          | None           |             |
| Sticky Note4 | Sticky Note         | (No content)                         | None          | None           |             |
| Sticky Note5 | Sticky Note         | (No content)                         | None          | None           |             |
| Sticky Note6 | Sticky Note         | (No content)                         | None          | None           |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**  
   - Type: Manual Trigger  
   - Purpose: To start the workflow manually.

2. **Add a Code node named "Get Date"**  
   - Type: Code (JavaScript)  
   - Purpose: Retrieve the current date/time. For example, use:  
     ```javascript
     return [{ json: { currentDate: new Date().toISOString() } }];
     ```  
   - Connect: Manual Trigger → Get Date

3. **Add a Notion node**  
   - Type: Notion  
   - Purpose: Fetch client data from your Notion database.  
   - Configuration:  
     - Authenticate with Notion credentials.  
     - Set the database ID or page to query.  
     - Optionally, set filters or sorts as appropriate.  
   - Connect: Get Date → Notion

4. **Add a Code node named "Calculate Days"**  
   - Type: Code (JavaScript)  
   - Purpose: Calculate days elapsed between a client date field from Notion and the current date.  
   - Sample logic:  
     ```javascript
     const currentDate = new Date(items[0].json.currentDate);
     return items.map(item => {
       const clientDate = new Date(item.json.clientDateField); // replace clientDateField with actual field name
       const diffTime = currentDate - clientDate;
       const diffDays = Math.floor(diffTime / (1000 * 60 * 60 * 24));
       item.json.daysElapsed = diffDays;
       return item;
     });
     ```  
   - Connect: Notion → Calculate Days

5. **Add three If nodes named "Day 7", "Day 30", and "Day 60"**  
   - Type: If  
   - Purpose: Check if `daysElapsed` equals 7, 30, or 60 respectively.  
   - Configuration for each:  
     - Condition: `daysElapsed` equals 7 (or 30, or 60)  
   - Connect: Calculate Days → Day 7, Day 30, Day 60 (each in parallel)

6. **Add Email Send nodes named "Email 7", "Email 30", and "Email 60"**  
   - Type: Email Send  
   - Purpose: Send nurture emails tailored for each milestone.  
   - Configuration:  
     - Set SMTP or other email credentials.  
     - Configure recipient email dynamically from Notion client data.  
     - Define subject and body content specific for day 7, 30, or 60.  
   - Connect:  
     - Day 7 (true) → Email 7  
     - Day 30 (true) → Email 30  
     - Day 60 (true) → Email 60

7. **Save and test the workflow**  
   - Trigger manually.  
   - Verify correct email dispatch at correct day intervals.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                            |
|-----------------------------------------------------------------------------------------------------------|------------------------------------------------------------|
| This workflow relies heavily on accurate date fields in Notion; ensure client date properties are consistent and correctly formatted. | Workflow requirement                                        |
| Email nodes require proper SMTP or email service credentials to be configured in n8n for successful sending. | Credential configuration                                    |
| Consider adding error handling or retry mechanisms on API calls and email dispatch to improve robustness. | Best practices for production workflows                     |
| For advanced conditional logic or dynamic email content, integrate additional code or template nodes.   | Possible workflow extension                                 |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.
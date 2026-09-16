Audit inactive Microsoft 365 users and email license reports via Outlook

https://n8nworkflows.xyz/workflows/audit-inactive-microsoft-365-users-and-email-license-reports-via-outlook-18524


# Audit inactive Microsoft 365 users and email license reports via Outlook

### 1. Workflow Overview

This workflow automates the monthly audit of Microsoft 365 environments to identify inactive users assigned to paid licenses. Its primary purpose is to help IT and finance teams reclaim wasted software costs by detecting accounts that have remained inactive past a configured threshold. 

The execution logic is structured into four sequential blocks:
- **1.1 Initialization and Data Retrieval:** Triggers on a monthly schedule, establishes configuration parameters, and queries the Microsoft Graph API to retrieve tenant subscription SKUs and paginated user accounts with sign-in logs.
- **1.2 Filtering and Analysis:** Processes the raw user data, identifies active versus inactive licensed accounts based on inactivity thresholds, maps assigned SKUs to friendly names, and computes estimated monthly cost savings.
- **1.3 Conditional Branching and Report Generation:** Evaluates whether inactive users exist. If found, it formats an Excel workbook and compiles an HTML summary containing financial metrics and top cost-incurring accounts.
- **1.4 Notification Delivery:** Dispatches either an HTML email containing the attached Excel report to the designated recipient or an all-clear notice if no inactive users are detected.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Initialization and Data Retrieval
- **Overview:** Initializes the auditing cycle on a scheduled basis, provisions environment configurations, and fetches enterprise license pricing alongside user account datasets from Microsoft Graph.
- **Nodes Involved:** 
  - `When Monthly on 1st at 7AM`
  - `Set Audit Configuration`
  - `Fetch Subscribed SKUs`
  - `Fetch All Users Paginated`
- **Node Details:**
  - **When Monthly on 1st at 7AM**
    - *Type & Role:* Schedule Trigger node executing the workflow automatically on the first day of every month at 07:00 AM.
    - *Configuration:* Standard cron-like scheduling rules configured for monthly recurrence.
    - *Connections:* Output connects directly to `Set Audit Configuration`.
    - *Edge Cases:* Server downtime during the exact trigger window may cause execution skips until the next cycle.
  - **Set Audit Configuration**
    - *Type & Role:* Set node defining global workflow variables such as inactivity thresholds, target recipient emails, tenant names, and cost mapping dictionaries.
    - *Configuration:* Sets parameters including `thresholdDays`, `reportRecipient`, `companyName`, and `licensePrices`.
    - *Connections:* Input from `When Monthly on 1st at 7AM`; output connects to `Fetch Subscribed SKUs`.
    - *Edge Cases:* Invalid JSON structures in the price map variables can break subsequent code node transformations.
  - **Fetch Subscribed SKUs**
    - *Type & Role:* HTTP Request node querying the Microsoft Graph API (`/v1.0/subscribedSkus`) to retrieve active subscription definitions.
    - *Configuration:* Uses Microsoft Graph OAuth2 credentials, enables retry on failure up to 3 attempts with a 5000ms delay.
    - *Connections:* Input from `Set Audit Configuration`; output connects to `Fetch All Users Paginated`.
    - *Version Requirements:* TypeVersion 4.2.
    - *Edge Cases:* Expired OAuth2 tokens or insufficient permissions (`Directory.Read.All`) will trigger authentication errors.
  - **Fetch All Users Paginated**
    - *Type & Role:* HTTP Request node querying Microsoft Graph (`/v1.0/users`) with expansion logic for sign-in activity and assigned licenses.
    - *Configuration:* Configured for pagination traversal, maximum 3 retries with 5000ms backoff on failures.
    - *Connections:* Input from `Fetch Subscribed SKUs`; output connects to `Filter Inactive Users & Estimate Costs`.
    - *Edge Cases:* Large enterprise tenants may experience rate-limiting (HTTP 429) requiring robust retry-handling configurations.

#### Block 1.2: Filtering and Analysis
- **Overview:** Evaluates user sign-in timestamps against the inactivity threshold, filters out active accounts, maps raw SKU IDs to business names, and calculates total reclaimable costs.
- **Nodes Involved:**
  - `Filter Inactive Users & Estimate Costs`
- **Node Details:**
  - **Filter Inactive Users & Estimate Costs**
    - *Type & Role:* Code node running JavaScript/Python to perform data wrangling, date arithmetic, and aggregation calculations.
    - *Configuration:* Iterates through user lists, compares `lastSignInDateTime` against `thresholdDays`, and applies pricing models from configuration datasets.
    - *Connections:* Input from `Fetch All Users Paginated`; output connects to `Check for Inactive Users`.
    - *Edge Cases:* Users with null or missing sign-in logs must be handled safely to avoid runtime exceptions.

#### Block 1.3: Conditional Branching and Report Generation
- **Overview:** Splits workflow execution depending on whether inactive users were found, formatting tabular spreadsheet data and HTML summaries for reporting.
- **Nodes Involved:**
  - `Check for Inactive Users`
  - `Build Excel Report`
  - `Generate HTML Summary`
- **Node Details:**
  - **Check for Inactive Users**
    - *Type & Role:* IF node evaluating whether the filtered dataset contains one or more inactive user records.
    - *Configuration:* Conditional evaluation based on array length or item presence from the preceding code node.
    - *Connections:* Input from `Filter Inactive Users & Estimate Costs`; true branch outputs to `Build Excel Report`, false branch outputs to `Send All-Clear Email Notice`.
  - **Build Excel Report**
    - *Type & Role:* Convert to File node compiling structured user datasets into an `.xlsx` binary file attachment.
    - *Configuration:* Configured to output Excel format with designated headers mapping user attributes and cost estimates.
    - *Connections:* Input from `Check for Inactive Users` (true branch); output connects to `Generate HTML Summary`.
  - **Generate HTML Summary**
    - *Type & Role:* Code node crafting a responsive HTML email body detailing cost metrics and top candidates for license reclamation.
    - *Configuration:* Injects variables like total estimated savings, company name, and counts into an HTML template string.
    - *Connections:* Input from `Build Excel Report`; output connects to `Email Inactive User Report`.

#### Block 1.4: Notification Delivery
- **Overview:** Delivers the audit outcomes via Microsoft Outlook to the designated administrator or reporting recipient.
- **Nodes Involved:**
  - `Email Inactive User Report`
  - `Send All-Clear Email Notice`
- **Node Details:**
  - **Email Inactive User Report**
    - *Type & Role:* Microsoft Outlook node sending an email containing the generated HTML body and attached Excel spreadsheet.
    - *Configuration:* Utilizes Outlook OAuth2 credentials, supports file attachments and HTML message formatting. Configured with retry logic (3 attempts, 5000ms delay).
    - *Connections:* Input from `Generate HTML Summary`.
    - *Edge Cases:* Mailbox storage quotas, recipient address typos, or missing mail-send permissions (`Mail.Send`).
  - **Send All-Clear Email Notice**
    - *Type & Role:* Microsoft Outlook node sending a notification email when zero inactive licenses are identified during the audit period.
    - *Configuration:* Sends a clean text or HTML status notice using Outlook credentials.
    - *Connections:* Input from `Check for Inactive Users` (false branch).
    - *Edge Cases:* Same authentication and permission requirements as the primary notification node.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| When Monthly on 1st at 7AM | scheduleTrigger | Triggers audit workflow monthly | None | Set Audit Configuration | |
| Set Audit Configuration | set | Defines thresholds, recipients, and pricing | When Monthly on 1st at 7AM | Fetch Subscribed SKUs | |
| Fetch Subscribed SKUs | httpRequest | Retrieves tenant licenses from Graph API | Set Audit Configuration | Fetch All Users Paginated | |
| Fetch All Users Paginated | httpRequest | Fetches paginated user data and sign-ins | Fetch Subscribed SKUs | Filter Inactive Users & Estimate Costs | |
| Filter Inactive Users & Estimate Costs | code | Identifies inactive users and calculates costs | Fetch All Users Paginated | Check for Inactive Users | |
| Check for Inactive Users | if | Branches workflow based on findings | Filter Inactive Users & Estimate Costs | Build Excel Report, Send All-Clear Email Notice | |
| Build Excel Report | convertToFile | Generates Excel file of inactive users | Check for Inactive Users | Generate HTML Summary | |
| Generate HTML Summary | code | Creates HTML email body summary | Build Excel Report | Email Inactive User Report | |
| Email Inactive User Report | microsoftOutlook | Emails report and Excel attachment | Generate HTML Summary | None | |
| Send All-Clear Email Notice | microsoftOutlook | Sends all-clear email if no inactive users | Check for Inactive Users | None | |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger:**
   - Add a **Schedule Trigger** node (`When Monthly on 1st at 7AM`).
   - Configure recurrence to run on the 1st of every month at 07:00 AM.
2. **Configure Global Variables:**
   - Add a **Set** node (`Set Audit Configuration`).
   - Define parameters including `thresholdDays` (e.g., 90), `reportRecipient` (email address), `companyName`, and a JSON map for `licensePrices`.
3. **Fetch Tenant Subscriptions:**
   - Add an **HTTP Request** node (`Fetch Subscribed SKUs`).
   - Set method to `GET`, URL to `https://graph.microsoft.com/v1.0/subscribedSkus`.
   - Configure Microsoft Graph OAuth2 credentials with `Directory.Read.All` permissions. Enable retry behavior (3 attempts, 5000ms wait).
4. **Retrieve M365 User Accounts:**
   - Add an **HTTP Request** node (`Fetch All Users Paginated`).
   - Set method to `GET`, URL to `https://graph.microsoft.com/v1.0/users?$select=id,displayName,userPrincipalName,accountEnabled,assignedLicenses&$expand=signInActivity`.
   - Enable pagination options and configure retry rules. Connect input from `Fetch Subscribed SKUs`.
5. **Process and Filter Inactive Accounts:**
   - Add a **Code** node (`Filter Inactive Users & Estimate Costs`).
   - Write JavaScript logic to compare sign-in timestamps against the threshold, cross-reference assigned SKUs, and sum estimated monthly savings. Connect input from `Fetch All Users Paginated`.
6. **Evaluate Audit Results:**
   - Add an **IF** node (`Check for Inactive Users`).
   - Configure condition to verify if the inactive user list length is greater than zero. Connect input from the Code node.
7. **Build Report Artifacts (True Branch):**
   - Connect the *True* output to a **Convert to File** node (`Build Excel Report`) configured to convert JSON items into an `.xlsx` binary stream.
   - Connect the Excel node output to a **Code** node (`Generate HTML Summary`) to format the metrics and top candidates into an HTML string.
8. **Dispatch Audit Report:**
   - Connect the HTML summary output to a **Microsoft Outlook** node (`Email Inactive User Report`).
   - Configure Microsoft Outlook OAuth2 credentials, set recipient variables, attach the binary file from the Excel node, and set the message body to HTML.
9. **Dispatch All-Clear Notice (False Branch):**
   - Connect the *False* output of the IF node to a second **Microsoft Outlook** node (`Send All-Clear Email Notice`).
   - Configure recipient parameters and compose a standard notification confirming zero inactive licensed users were discovered.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Microsoft Graph API Documentation | [Microsoft Graph API Reference](https://learn.microsoft.com/en-us/graph/overview) |
| n8n Microsoft Outlook Integration Guide | [n8n Outlook Node Docs](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.microsoftoutlook/) |
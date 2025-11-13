Detect Unused Android Feature Flags with GitLab, LaunchDarkly, Jira & Slack

https://n8nworkflows.xyz/workflows/detect-unused-android-feature-flags-with-gitlab--launchdarkly--jira---slack-7985


# Detect Unused Android Feature Flags with GitLab, LaunchDarkly, Jira & Slack

### 1. Workflow Overview

This workflow automates the detection of unused ("dead") Android feature flags in a Kotlin/Java codebase by comparing the flags found in the source code repository with the current feature flags and their status in LaunchDarkly. It operates on a weekly schedule and integrates with Google Sheets for reporting, Jira for issue tracking, and Slack for team notifications. The purpose is to help reduce technical debt by identifying and acting on obsolete feature flags.

**Logical blocks:**

- **1.1 Scheduled Trigger & Source Code Analysis**: Weekly trigger initiates the process, fetches Android code from GitLab, and extracts feature flags used in the code.
- **1.2 LaunchDarkly Flags Retrieval & Dead Flag Detection**: Fetches all feature flags from LaunchDarkly and compares them with the code flags to identify dead flags.
- **1.3 Reporting and Notifications**: Stores dead flags in Google Sheets, then if dead flags exist, creates Jira issues and sends Slack alerts.
- **1.4 Conditional Execution**: Checks if dead flags are found before proceeding to Jira and Slack notifications.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Scheduled Trigger & Source Code Analysis

- **Overview:**  
  This block initiates the workflow weekly, fetches the Android Kotlin/Java source code from GitLab, and extracts the feature flags used in the codebase by searching for a specific naming pattern.

- **Nodes Involved:**  
  - Schedule Trigger  
  - GitLab  
  - Detect flags (Code node)

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Starts workflow execution weekly at 10 AM (UTC assumed)  
    - Configuration: Interval set to every 1 week, trigger hour 10  
    - Inputs: None (entry point)  
    - Outputs: Triggers GitLab node  
    - Edge cases: Missed trigger if n8n server downtime at scheduled time  

  - **GitLab**  
    - Type: GitLab node  
    - Role: Fetches relevant Android source code repository content to analyze flags  
    - Configuration: Repository owner, name, resource, and operation need to be specified (currently empty placeholders)  
    - Authentication: Uses OAuth2 credentials for GitLab API  
    - Inputs: Trigger from Schedule Trigger  
    - Outputs: Passes fetched code content to "Detect flags" node  
    - Edge cases: Authentication failure, rate limiting, repository access issues, empty or malformed file content  

  - **Detect flags**  
    - Type: Code (JavaScript)  
    - Role: Parses the fetched code files to extract feature flags matching the pattern `ENABLE_[A-Z0-9_]+`  
    - Configuration: Uses regex to find all occurrences of the pattern in the content field of input data and returns unique flags  
    - Expressions/Variables:  
      - Uses `$input.all()` to process all incoming files  
      - Extracts `content` JSON property from each input item  
      - Outputs an array of JSON objects each containing `{ flag: <flag_name> }`  
    - Inputs: Content from GitLab node  
    - Outputs: Passes extracted flags to HTTP Request node for LaunchDarkly API call  
    - Edge cases: Files missing `content` property, regex failing, no flags found  

---

#### 2.2 LaunchDarkly Flags Retrieval & Dead Flag Detection

- **Overview:**  
  Retrieves all feature flags from LaunchDarkly via API, then compares them against the flags detected in the codebase to identify dead flags — those not used in code and either archived or off in production.

- **Nodes Involved:**  
  - HTTP Request  
  - Find dead flags (Code node)  

- **Node Details:**

  - **HTTP Request**  
    - Type: HTTP Request  
    - Role: Calls LaunchDarkly API endpoint `/api/v2/flags/default` to fetch all flags for the default environment  
    - Configuration:  
      - URL set to LaunchDarkly flags API  
      - Headers configured to include authorization or required API keys (currently empty placeholders)  
      - Sends headers, no body specified  
    - Inputs: Output of "Detect flags" node (flags extracted from code)  
    - Outputs: LaunchDarkly flags JSON response to "Find dead flags" node  
    - Edge cases: API auth errors, rate limiting, network timeouts, malformed JSON response  

  - **Find dead flags**  
    - Type: Code (JavaScript)  
    - Role: Filters the LaunchDarkly flags to find those that are unused in code and either archived or off in production environment  
    - Configuration Logic:  
      - Extracts `items` array from LaunchDarkly API response  
      - Reads flags extracted from code from the "Detect flags" node via `$('Detect flags')`  
      - Filters flags by checking:  
        - Flag key not included in code flags  
        - Flag is either archived in both production and test environments or turned off in production  
      - Returns array of JSON objects containing dead flag keys and reasons ("Archived" or "Off in Production")  
    - Inputs: LaunchDarkly API response and detected code flags  
    - Outputs: Dead flags to Google Sheets node  
    - Edge cases: Unexpected API response shape, missing environments data, empty inputs  

---

#### 2.3 Reporting and Notifications

- **Overview:**  
  Saves the identified dead flags into a Google Sheet for reporting, and if dead flags exist, creates Jira issues and sends Slack notifications to alert the team.

- **Nodes Involved:**  
  - Google Sheets  
  - Check Dead Flags (If node)  
  - Jira Software  
  - Slack  

- **Node Details:**

  - **Google Sheets**  
    - Type: Google Sheets  
    - Role: Appends or updates the Google Sheet with dead flags data  
    - Configuration:  
      - Document ID and Sheet Name to be specified (currently placeholders)  
      - Uses Google OAuth2 credentials  
    - Inputs: Dead flags from "Find dead flags" node  
    - Outputs: Passes data to "Check Dead Flags" node  
    - Edge cases: Authentication failure, sheet access permissions, invalid document or sheet name  

  - **Check Dead Flags**  
    - Type: If node  
    - Role: Checks if any dead flags were found by verifying `deadFlag` property presence in input JSON  
    - Configuration: Condition tests if `deadFlag` is NOT undefined (i.e., dead flags exist)  
    - Inputs: Output from Google Sheets node  
    - Outputs: If true (dead flags found), proceeds to Jira and Slack nodes; otherwise, workflow ends  
    - Edge cases: Input data structure issues, false negatives if property missing or malformed  

  - **Jira Software**  
    - Type: Jira node  
    - Role: Creates Jira issues for each dead flag found to track removal or investigation efforts  
    - Configuration:  
      - Project and Issue Type to be specified via dynamic list  
      - Summary field dynamically set to "Dead feature flag: <flag_name>"  
      - Uses Jira API credentials  
    - Inputs: Dead flags filtered by "Check Dead Flags" node  
    - Outputs: None (end node)  
    - Edge cases: Authentication errors, project or issue type misconfiguration, API limits  

  - **Slack**  
    - Type: Slack node  
    - Role: Sends Slack alerts notifying the team about detected dead feature flags  
    - Configuration:  
      - Uses a configured Slack Webhook ID  
      - Message content dynamically constructed (not shown in JSON, presumably includes dead flag info)  
    - Inputs: Dead flags filtered by "Check Dead Flags" node  
    - Outputs: None (end node)  
    - Edge cases: Webhook misconfiguration, network errors, rate limiting  

---

#### 2.4 Conditional Execution

- **Overview:**  
  Ensures that Jira and Slack actions are performed only if dead flags are actually found, preventing unnecessary notifications.

- **Nodes Involved:**  
  - Check Dead Flags (If node)  

- **Node Details:**  
  - See details above in 2.3 for the Check Dead Flags node.

---

### 3. Summary Table

| Node Name          | Node Type            | Functional Role                                  | Input Node(s)     | Output Node(s)    | Sticky Note                                                                                                     |
|--------------------|----------------------|-------------------------------------------------|-------------------|-------------------|-----------------------------------------------------------------------------------------------------------------|
| Schedule Trigger    | Schedule Trigger     | Initiates workflow weekly                        | None              | GitLab            |                                                                                                                 |
| GitLab             | GitLab               | Fetch Android Kotlin/Java source code            | Schedule Trigger  | Detect flags      |                                                                                                                 |
| Detect flags       | Code                 | Extract feature flags from code content          | GitLab            | HTTP Request      |                                                                                                                 |
| HTTP Request       | HTTP Request         | Retrieve all feature flags from LaunchDarkly API | Detect flags      | Find dead flags   |                                                                                                                 |
| Find dead flags    | Code                 | Identify unused dead flags by comparing lists    | HTTP Request      | Google Sheets     |                                                                                                                 |
| Google Sheets      | Google Sheets        | Store dead flags data                             | Find dead flags   | Check Dead Flags  |                                                                                                                 |
| Check Dead Flags   | If                   | Conditional check if dead flags exist            | Google Sheets     | Jira Software, Slack |                                                                                                                 |
| Jira Software      | Jira                 | Create Jira issues for dead flags                 | Check Dead Flags  | None              |                                                                                                                 |
| Slack              | Slack                | Send Slack alerts about dead flags                | Check Dead Flags  | None              |                                                                                                                 |
| Sticky Note        | Sticky Note          | Overview and purpose of workflow                  | None              | None              | **Purpose:** Automatically find unused (“dead”) feature flags in an Android Kotlin/Java codebase and notify the team. Core Logic and Outcome described. |
| Sticky Note1       | Sticky Note          | Workflow title banner                             | None              | None              | ## Catalog Android (Kotlin/Java) feature flags and sweep dead flags vs LaunchDarkly (weekly) → Sheets + Jira + PR + Slack |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**:  
   - Set to trigger every 1 week at hour 10 (e.g., every Monday 10 AM UTC).  
   - This node is the workflow entry point.

2. **Add a GitLab node**:  
   - Configure it to access your Android Kotlin/Java repository.  
   - Set repository owner, name, resource (e.g., files or repository tree), and operation (e.g., get file content or list files).  
   - Authenticate using GitLab OAuth2 credentials.  
   - Connect Schedule Trigger → GitLab.

3. **Add a Code node named "Detect flags"**:  
   - Use JavaScript to scan through the fetched code files' content.  
   - Use regex pattern `/ENABLE_[A-Z0-9_]+/g` to extract all unique feature flags.  
   - Output each found flag as `{ json: { flag: <flag_name> } }`.  
   - Connect GitLab → Detect flags.

4. **Add an HTTP Request node**:  
   - Configure it to call LaunchDarkly API endpoint: `https://app.launchdarkly.com/api/v2/flags/default`.  
   - Include any necessary authentication headers (e.g., API key) under header parameters.  
   - Set HTTP method to GET.  
   - Connect Detect flags → HTTP Request.

5. **Add a Code node named "Find dead flags"**:  
   - JavaScript logic:  
     - Extract all flags from LaunchDarkly response (`items` array).  
     - Retrieve flags detected in code from "Detect flags" node.  
     - Filter LaunchDarkly flags where flag key is NOT in code flags AND is either archived in both production and test environments OR turned off in production.  
     - Return JSON array with dead flags and reason.  
   - Connect HTTP Request → Find dead flags.

6. **Add a Google Sheets node**:  
   - Configure with the target Google Sheet document ID and sheet name where dead flags data will be stored.  
   - Authenticate using Google OAuth2 credentials.  
   - Connect Find dead flags → Google Sheets.

7. **Add an If node named "Check Dead Flags"**:  
   - Condition: Check if `deadFlag` property exists and is not undefined in the incoming JSON data.  
   - Connect Google Sheets → Check Dead Flags.

8. **Add a Jira node**:  
   - Configure with the target Jira project and issue type (e.g., task or bug).  
   - Use dynamic expression for Summary: `"Dead feature flag: {{$json.deadFlag}}"`.  
   - Authenticate with Jira credentials.  
   - Connect Check Dead Flags (True output) → Jira Software.

9. **Add a Slack node**:  
   - Configure with Slack Webhook ID.  
   - Set message content to notify about dead flags (customize as needed).  
   - Connect Check Dead Flags (True output) → Slack.

10. **Ensure nodes "Jira Software" and "Slack" are only triggered if dead flags exist** via the If node.

11. **Optionally, add Sticky Note nodes** to document workflow purpose and title.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                       | Context or Link                                                                                                                                                      |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| This workflow helps maintain clean feature flag usage in Android projects by automatically identifying and reporting unused flags, reducing tech debt and improving code quality. | Workflow purpose explanation (Sticky Note).                                                                                                                        |
| The regex used for flag detection targets flags with the pattern `ENABLE_` followed by uppercase letters, digits, or underscores, matching common Android feature flag naming.    | Important for adapting the code flag detection logic if your flags follow a different naming convention.                                                           |
| Jira and Slack integrations require proper API credentials and permissions configured in n8n to function correctly.                                                                | Credential setup instructions for GitLab OAuth2, Google Sheets OAuth2, Jira API, and Slack Webhook are essential before activation.                               |
| LaunchDarkly API calls require API keys with permission to read flags. Check the API documentation for authentication and endpoint details: https://docs.launchdarkly.com/api/v2/flags | Useful for troubleshooting API authentication or expanding the workflow with additional flag data if needed.                                                     |
| Google Sheets must have edit permissions granted to the associated OAuth2 user to append or update data.                                                                            | Ensure spreadsheet ID and worksheet name are correct to avoid runtime errors.                                                                                       |

---

**Disclaimer:**  
The provided text is exclusively derived from an n8n automated workflow. It complies strictly with content policies and contains no illegal, offensive, or protected material. All processed data is legal and public.
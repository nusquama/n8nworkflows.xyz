Monitor Workflow Audits and Failures with InfluxDB Dashboard

https://n8nworkflows.xyz/workflows/monitor-workflow-audits-and-failures-with-influxdb-dashboard-5873


# Monitor Workflow Audits and Failures with InfluxDB Dashboard

### 1. Workflow Overview

This n8n workflow, titled **"Monitor Workflow Audits and Failures with InfluxDB Dashboard"**, is designed to collect and send audit data and workflow execution metrics from an n8n instance to an InfluxDB time-series database. Its primary purpose is to enable monitoring and historical tracking of n8n audits and failures, which can then be visualized using an InfluxDB dashboard.

The workflow is structured into the following logical blocks:

- **1.1 Trigger and Global Variable Setup**  
  Handles manual or scheduled triggering and sets the global InfluxDB connection parameters.

- **1.2 Audits Collection**  
  Retrieves different audit reports from the n8n instance for categories like credentials, database, filesystem, instance, and nodes.

- **1.3 Audit Data Parsing and Formatting**  
  Parses audit data into InfluxDB line protocol format suitable for ingestion.

- **1.4 Workflow Execution Metrics Collection**  
  Gathers counts of active workflows and failed executions.

- **1.5 Metrics Formatting and Sending to InfluxDB**  
  Formats workflow execution metrics and audit data for InfluxDB and sends them via HTTP requests.

- **1.6 Notes and Documentation**  
  Provides explanatory sticky notes with instructions and best practices embedded inside the workflow.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and Global Variable Setup

**Overview:**  
This block manages workflow activation, either manually or scheduled once daily, and sets global InfluxDB connection parameters used throughout the workflow.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Once a Day (Schedule Trigger)  
- Influx Globals (Set)  
- Sticky Note2 (Sticky Note)  
- Sticky Note3 (Sticky Note)

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Allows manual execution of the workflow for on-demand runs.  
  - Input: None  
  - Output: Triggers the workflow start  
  - Edge Cases: None

- **Once a Day**  
  - Type: Schedule Trigger  
  - Role: Automatically triggers the workflow once daily at 05:00 (5 AM).  
  - Configuration: Interval trigger at hour 5 daily  
  - Input: None  
  - Output: Triggers workflow start  
  - Edge Cases: Timezone considerations may affect exact trigger time.

- **Influx Globals**  
  - Type: Set  
  - Role: Defines global variables for InfluxDB connection such as URL, organization, and bucket name.  
  - Configuration:  
    - influx_url: e.g., "https://example.com"  
    - influx_org: e.g., "myorg"  
    - influx_bucket: e.g., "n8n"  
  - Input: Trigger from manual or schedule node  
  - Output: Passes global variables downstream  
  - Edge Cases: Ensure correct InfluxDB URL and credentials; misconfiguration leads to connection failures.

- **Sticky Note2 & Sticky Note3**  
  - Type: Sticky Note  
  - Role: Provide user documentation about InfluxDB variables and scheduling recommendations.  
  - Content: Instructions on required InfluxDB variables and scheduling audit runs.

---

#### 2.2 Audits Collection

**Overview:**  
This block collects audit data from various categories in the n8n instance via the n8n API.

**Nodes Involved:**  
- Credentials Audit  
- Database Audit  
- Filesystem Audit  
- Instance Audit  
- Nodes Audit  
- Sticky Note1 (Sticky Note)

**Node Details:**

- **[Each Audit Node] (Credentials, Database, Filesystem, Instance, Nodes Audit)**  
  - Type: n8n API Node (Resource: Audit, Operation: Generate)  
  - Role: Requests audit reports for a specific category from the n8n instance.  
  - Configuration:  
    - Resource: audit  
    - Operation: generate  
    - Additional Options: categories set to the specific audit type (e.g., ["database"])  
  - Credentials: Uses n8n API credentials (OAuth2 or API key)  
  - Input: Trigger from "Influx Globals" node  
  - Output: Audit report JSON objects  
  - Edge Cases:  
    - API authentication errors  
    - Network timeouts  
    - Empty or malformed audit reports

- **Sticky Note1**  
  - Role: Explains the purpose of audits and their parsing for further processing.

---

#### 2.3 Audit Data Parsing and Formatting

**Overview:**  
This block extracts relevant sections and risk data from each audit report, splits report arrays, and formats the data into InfluxDB line protocol strings.

**Nodes Involved:**  
- Pull Report Data and Risk Credentials  
- Pull Report Data and Risk Database  
- Pull Report Data and Risk Filesystem  
- Pull Report Data and Risk Instance  
- Pull Report Data and Risk Nodes  
- Split Report Data  
- Prepare Influx Input Strings  
- Contactenate with Commas  
- Sticky Note (Parsing Instructions)

**Node Details:**

- **Pull Report Data and Risk [Category]** (five nodes)  
  - Type: Set  
  - Role: Extracts the `sections` array as `report` and `risk` string from each audit category's report JSON.  
  - Key Expressions:  
    - `report = {{ $json['<Category> Risk Report'].sections }}`  
    - `risk = {{ $json['<Category> Risk Report'].risk }}`  
  - Input: Output from corresponding Audit node  
  - Output: JSON with simplified fields for report and risk  
  - Edge Cases: Missing or empty fields in API response

- **Split Report Data**  
  - Type: SplitOut  
  - Role: Splits the `report` array into individual items for processing each report section separately.  
  - Parameters: Splits on field `report`, includes fields `risk` and first report title  
  - Input: Output from Pull Report Data nodes  
  - Output: Separate items per report section  
  - Edge Cases: Empty report arrays

- **Prepare Influx Input Strings**  
  - Type: Set  
  - Role: Constructs InfluxDB line protocol strings with keys and counts from report titles and locations.  
  - Expression:  
    - `title_key_count = {{ $json.report.title.toLowerCase().replaceAll(" ","_") }}={{ $json.report.location?.length ?? 0 }}`  
  - Input: Output from Split Report Data  
  - Output: JSON with formatted strings  
  - Edge Cases: Missing titles or location arrays

- **Contactenate with Commas**  
  - Type: Summarize  
  - Role: Concatenates all `title_key_count` strings using commas to form a single InfluxDB line.  
  - Input: Output from Prepare Influx Input Strings  
  - Output: Single concatenated string  
  - Edge Cases: No items to concatenate

- **Sticky Note** (Parsing Instructions)  
  - Role: Provides an example of InfluxDB line protocol data format and explains the parsing logic.

---

#### 2.4 Workflow Execution Metrics Collection

**Overview:**  
Collects and counts the number of active workflows and failed workflow executions from the n8n instance.

**Nodes Involved:**  
- Get Active Workflows  
- Get Failed Executions  
- Count Active  
- Count Failed  
- Merge Into One Reporting Line  
- Adjust Field Name  
- Sticky Note4 (Active Workflows and Failed Executions Explanation)

**Node Details:**

- **Get Active Workflows**  
  - Type: n8n API Node (Resource: n8n, Operation: List Workflows)  
  - Role: Retrieves all active workflows in the n8n instance.  
  - Filters: `activeWorkflows = true`  
  - Credentials: Uses n8n API credentials  
  - Input: Trigger from "Influx Globals"  
  - Output: List of active workflows  
  - Edge Cases: API errors, empty results

- **Get Failed Executions**  
  - Type: n8n API Node (Resource: execution, Operation: List Executions)  
  - Role: Retrieves all executions with status "error" (failed).  
  - Filters: `status = "error"`  
  - Return All: true  
  - Credentials: Uses n8n API credentials  
  - Input: Trigger from "Influx Globals"  
  - Output: List of failed executions  
  - Edge Cases: Large result sets may timeout

- **Count Active**  
  - Type: Summarize  
  - Role: Counts the number of active workflows.  
  - Field to summarize: `active` (assumed to be a count field from previous node)  
  - Input: From Get Active Workflows  
  - Output: Count JSON

- **Count Failed**  
  - Type: Summarize  
  - Role: Counts the number of failed executions.  
  - Field to summarize: `id` (counting failed executions by id)  
  - Input: From Get Failed Executions  
  - Output: Count JSON

- **Merge Into One Reporting Line**  
  - Type: Merge (Combine by Position)  
  - Role: Combines counts from active workflows and failed executions into one JSON object.  
  - Input: From Count Active and Count Failed nodes  
  - Output: Combined JSON with both counts

- **Adjust Field Name**  
  - Type: Set  
  - Role: Formats counts into a single InfluxDB line protocol string with keys `active_workflows` and `failed_executions`.  
  - Expression:  
    - `title_key_count = active_workflows={{ $json.count_active }},failed_executions={{ $json.count_id }}`  
  - Input: From Merge node  
  - Output: Single line protocol string for metrics

- **Sticky Note4**  
  - Role: Explains rationale for sending only counts of failed executions (grouped) rather than raw execution data.

---

#### 2.5 Metrics Formatting and Sending to InfluxDB

**Overview:**  
Sends the formatted audit and workflow execution metrics data to InfluxDB using HTTP POST requests with appropriate authentication headers.

**Nodes Involved:**  
- Send Workflows and Fails to InfluxDB  
- Send Audit to InfluxDB

**Node Details:**

- **Send Workflows and Fails to InfluxDB**  
  - Type: HTTP Request  
  - Role: Sends active workflow and failed execution counts as InfluxDB line protocol data.  
  - URL: Constructed from Influx Globals variables with endpoint `/api/v2/write` and parameters for org, bucket, and precision=ms  
  - Body:  
    - Format: `audit,operational=workflows_and_failed {{ $json.title_key_count }} {{ $now.toMillis() }}`  
  - Method: POST  
  - Headers:  
    - Accept: application/json  
  - Authentication: HTTP Header Auth using InfluxDB token credential  
  - Input: From Adjust Field Name node  
  - Output: HTTP response (usually status)  
  - Edge Cases: Authentication failures, network errors, malformed line protocol

- **Send Audit to InfluxDB**  
  - Type: HTTP Request  
  - Role: Sends concatenated audit risk data as InfluxDB line protocol data.  
  - URL: Same pattern as above  
  - Body:  
    - Format: `audit,risk={{ $('Split Report Data').first().json.risk }} {{ $json.concatenated_title_key_count }} {{ $now.toMillis() }}`  
  - Method: POST  
  - Headers and Authentication: Same as above  
  - Input: From Contactenate with Commas node  
  - Output: HTTP response  
  - Edge Cases: Risk data missing or empty, API errors

---

### 3. Summary Table

| Node Name                         | Node Type                 | Functional Role                         | Input Node(s)                      | Output Node(s)                              | Sticky Note                                                                                           |
|----------------------------------|---------------------------|---------------------------------------|----------------------------------|---------------------------------------------|-----------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger            | Manual workflow start                  | None                             | Influx Globals                               |                                                                                                     |
| Once a Day                       | Schedule Trigger          | Scheduled workflow start (daily 5 AM) | None                             | Influx Globals                               | ## Schedule Audits Audits don't need to be run often, but I would recommend it to be run on regular basis. This way you can see real data series in InfluxDB. I think that once a day should be enough, but it depends on your N8N usage of course |
| Influx Globals                   | Set                       | Define InfluxDB connection parameters | Once a Day, Manual Trigger       | Credentials Audit, Nodes Audit, Database Audit, Filesystem Audit, Instance Audit, Get Failed Executions, Get Active Workflows | ## Influx Vars You will need 3 variables: - InfluxDB URL - Organization - Bucket name Put it in here |
| Credentials Audit                | n8n API (Audit Generate)  | Get credentials audit report          | Influx Globals                  | Pull Report Data and Risk Credentials        | ## Get N8N Audits Get different type of audits and parse them into format that will be understandable for further processing |
| Database Audit                  | n8n API (Audit Generate)  | Get database audit report             | Influx Globals                  | Pull Report Data and Risk Database           | ## Get N8N Audits Get different type of audits and parse them into format that will be understandable for further processing |
| Filesystem Audit                | n8n API (Audit Generate)  | Get filesystem audit report           | Influx Globals                  | Pull Report Data and Risk Filesystem         | ## Get N8N Audits Get different type of audits and parse them into format that will be understandable for further processing |
| Instance Audit                 | n8n API (Audit Generate)  | Get instance audit report             | Influx Globals                  | Pull Report Data and Risk Instance           | ## Get N8N Audits Get different type of audits and parse them into format that will be understandable for further processing |
| Nodes Audit                   | n8n API (Audit Generate)  | Get nodes audit report                | Influx Globals                  | Pull Report Data and Risk Nodes              | ## Get N8N Audits Get different type of audits and parse them into format that will be understandable for further processing |
| Pull Report Data and Risk Credentials | Set                   | Extract credentials audit report data | Credentials Audit               | Split Report Data                            |                                                                                                     |
| Pull Report Data and Risk Database | Set                     | Extract database audit report data    | Database Audit                  | Split Report Data                            |                                                                                                     |
| Pull Report Data and Risk Filesystem | Set                   | Extract filesystem audit report data  | Filesystem Audit                | Split Report Data                            |                                                                                                     |
| Pull Report Data and Risk Instance | Set                     | Extract instance audit report data    | Instance Audit                  | Split Report Data                            |                                                                                                     |
| Pull Report Data and Risk Nodes | Set                       | Extract nodes audit report data       | Nodes Audit                    | Split Report Data                            |                                                                                                     |
| Split Report Data              | SplitOut                   | Split report sections for processing  | Pull Report Data and Risk *     | Prepare Influx Input Strings                  | ## Parse for Influx-understandable data Influx sample data sent via CURL is following: ...            |
| Prepare Influx Input Strings   | Set                       | Format each report section into InfluxDB line protocol fields | Split Report Data              | Contactenate with Commas                      | ## Parse for Influx-understandable data Influx sample data sent via CURL is following: ...            |
| Contactenate with Commas       | Summarize                 | Concatenate formatted strings into one line | Prepare Influx Input Strings   | Send Audit to InfluxDB                        | ## Parse for Influx-understandable data Influx sample data sent via CURL is following: ...            |
| Get Active Workflows           | n8n API (List Workflows)   | Retrieve active workflows count       | Influx Globals                  | Count Active                                 | ## Active Worfklows and Failed Executions If you are feeling advanterous, you can also send data about workflows and executions to Influx. Since there can be a lot of executions, keep in mind to rather count one group ("failed") in this example |
| Get Failed Executions          | n8n API (List Executions)  | Retrieve failed executions count      | Influx Globals                  | Count Failed                                 | ## Active Worfklows and Failed Executions If you are feeling advanterous, you can also send data about workflows and executions to Influx. Since there can be a lot of executions, keep in mind to rather count one group ("failed") in this example |
| Count Active                  | Summarize                 | Count active workflows                 | Get Active Workflows            | Merge Into One Reporting Line                 | ## Active Worfklows and Failed Executions If you are feeling advanterous, you can also send data about workflows and executions to Influx. Since there can be a lot of executions, keep in mind to rather count one group ("failed") in this example |
| Count Failed                  | Summarize                 | Count failed executions                | Get Failed Executions           | Merge Into One Reporting Line                 | ## Active Worfklows and Failed Executions If you are feeling advanterous, you can also send data about workflows and executions to Influx. Since there can be a lot of executions, keep in mind to rather count one group ("failed") in this example |
| Merge Into One Reporting Line | Merge                     | Combine counts into one JSON           | Count Active, Count Failed      | Adjust Field Name                             | ## Active Worfklows and Failed Executions If you are feeling advanterous, you can also send data about workflows and executions to Influx. Since there can be a lot of executions, keep in mind to rather count one group ("failed") in this example |
| Adjust Field Name             | Set                       | Format counts into InfluxDB line protocol string | Merge Into One Reporting Line | Send Workflows and Fails to InfluxDB          | ## Active Worfklows and Failed Executions If you are feeling advanterous, you can also send data about workflows and executions to Influx. Since there can be a lot of executions, keep in mind to rather count one group ("failed") in this example |
| Send Workflows and Fails to InfluxDB | HTTP Request           | Send workflow metrics to InfluxDB     | Adjust Field Name              | None                                         |                                                                                                     |
| Send Audit to InfluxDB        | HTTP Request              | Send audit data to InfluxDB            | Contactenate with Commas       | None                                         |                                                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes**  
   - Add a **Manual Trigger** node named "When clicking ‘Execute workflow’". No parameters needed.  
   - Add a **Schedule Trigger** node named "Once a Day". Set the interval to daily trigger at hour 5 (5 AM).

2. **Create Global Variables Node**  
   - Add a **Set** node named "Influx Globals".  
   - Add three string fields:  
     - `influx_url` with your InfluxDB HTTP API URL (e.g., `https://example.com`).  
     - `influx_org` with your InfluxDB organization name (e.g., `myorg`).  
     - `influx_bucket` with your InfluxDB bucket name (e.g., `n8n`).  
   - Connect both trigger nodes ("When clicking ‘Execute workflow’" and "Once a Day") to this node.

3. **Add Audit Nodes**  
   For each audit category below, add an **n8n API** node with:  
   - Resource: `audit`  
   - Operation: `generate`  
   - Additional Options > Categories: `[category]` (replace `[category]` with one of: `credentials`, `database`, `filesystem`, `instance`, `nodes`)  
   - Credentials: Use your n8n API authentication (OAuth2 or API key).  
   - Connect "Influx Globals" node output to all these audit nodes.

4. **Extract Report and Risk Data**  
   For each audit node, add a **Set** node named "Pull Report Data and Risk [Category]" with:  
   - Assignments:  
     - `report` (type array): `={{ $json['[Category] Risk Report'].sections }}`  
     - `risk` (type string): `={{ $json['[Category] Risk Report'].risk }}`  
   - Connect each audit node to its corresponding Set node.

5. **Split Report Data**  
   - Add a **SplitOut** node named "Split Report Data" with:  
     - Field to split out: `report`  
     - Fields to include: `risk`, and `report[0].title`  
   - Connect all "Pull Report Data and Risk [Category]" nodes to this Split node.

6. **Prepare Influx Input Strings**  
   - Add a **Set** node "Prepare Influx Input Strings" with assignment:  
     - `title_key_count` (string): `={{ $json.report.title.toLowerCase().replaceAll(" ","_") }}={{ $json.report.location?.length ?? 0 }}`  
   - Connect "Split Report Data" node output here.

7. **Concatenate Strings**  
   - Add a **Summarize** node "Contactenate with Commas" to concatenate all `title_key_count` fields with commas.  
   - Connect "Prepare Influx Input Strings" output to this node.

8. **Add Workflow and Execution Metrics Nodes**  
   - Add **n8n API** node "Get Active Workflows" with:  
     - Filters: `activeWorkflows = true`  
     - Credentials: n8n API credentials  
     - Connect from "Influx Globals".  
   - Add **n8n API** node "Get Failed Executions" with:  
     - Filters: `status = "error"`  
     - Return all: true  
     - Credentials: n8n API credentials  
     - Connect from "Influx Globals".

9. **Count Metrics**  
   - Add **Summarize** node "Count Active" to count active workflows (field: `active`). Connect from "Get Active Workflows".  
   - Add **Summarize** node "Count Failed" to count failed executions (field: `id`). Connect from "Get Failed Executions".

10. **Merge Counts**  
    - Add **Merge** node "Merge Into One Reporting Line" with mode “Combine by Position”. Connect both "Count Active" and "Count Failed" here.

11. **Format Metrics for InfluxDB**  
    - Add **Set** node "Adjust Field Name" with assignment:  
      - `title_key_count` (string): `=active_workflows={{ $json.count_active }},failed_executions={{ $json.count_id }}`  
    - Connect "Merge Into One Reporting Line" here.

12. **Send Data to InfluxDB**  
    - Add **HTTP Request** node "Send Workflows and Fails to InfluxDB" with:  
      - URL: `={{ $('Influx Globals').item.json.influx_url }}/api/v2/write?org={{ $('Influx Globals').item.json.influx_org }}&bucket={{ $('Influx Globals').item.json.influx_bucket }}&precision=ms`  
      - Method: POST  
      - Body: `=audit,operational=workflows_and_failed {{ $json.title_key_count }} {{ $now.toMillis() }}`  
      - Content-Type: `text/plain; charset=utf-8`  
      - Authentication: HTTP Header Auth with Influx Token credential, header name "Authorization", value `Token <your_token>`  
      - Connect from "Adjust Field Name".

    - Add **HTTP Request** node "Send Audit to InfluxDB" with similar settings, but body:  
      - `=audit,risk={{ $('Split Report Data').first().json.risk }} {{ $json.concatenated_title_key_count }} {{ $now.toMillis() }}`  
      - Connect from "Contactenate with Commas".

13. **Add Sticky Notes**  
    - Add sticky notes near respective nodes providing explanations about scheduling, InfluxDB variables, audit parsing, and metrics rationale as per the original.

14. **Test and Activate**  
    - Test the workflow manually and on schedule.  
    - Ensure correct credentials and network connectivity to InfluxDB.  
    - Activate the workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                             | Context or Link                                                                                                                                 |
|------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------|
| Sample InfluxDB data format for line protocol and cURL example included in sticky note for parsing guidance.                            | https://docs.influxdata.com/influxdb/v2.0/write-data/developer-tools/api/#write-data                                                           |
| Recommendation: Schedule audits once daily to balance data freshness and system load.                                                   | Workflow Sticky Note3                                                                                                                           |
| To send workflow execution metrics, count groups (e.g., failed executions) rather than sending all execution data to avoid overload.   | Sticky Note4                                                                                                                                   |
| Requires valid InfluxDB token with write permissions to the specified bucket.                                                           | InfluxDB documentation: https://docs.influxdata.com/influxdb/v2.0/security/tokens/                                                              |
| n8n API credentials must have sufficient permissions to query audits, workflows, and executions.                                         | n8n documentation: https://docs.n8n.io/integrations/builtin/core-nodes/n8n/                                                                      |

---

*Disclaimer:* The provided content is extracted exclusively from an automated n8n workflow and complies fully with content policies. All data manipulated is legal and public.
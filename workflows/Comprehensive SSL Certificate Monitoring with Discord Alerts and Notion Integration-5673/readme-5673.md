Comprehensive SSL Certificate Monitoring with Discord Alerts and Notion Integration

https://n8nworkflows.xyz/workflows/comprehensive-ssl-certificate-monitoring-with-discord-alerts-and-notion-integration-5673


# Comprehensive SSL Certificate Monitoring with Discord Alerts and Notion Integration

### 1. Workflow Overview

This workflow is a **Comprehensive SSL Certificate Monitoring** system designed for DevOps engineers, IT administrators, and security professionals. It performs daily automated checks on multiple domains’ SSL certificates, cross-validates results using dual methods, and sends alerts via Discord when SSL issues are detected. The workflow integrates data sourcing from Notion, external SSL verification APIs, a custom SSL health scanner via SSH, and produces rich, actionable notifications in Discord.

**Target Use Cases:**
- Automated daily SSL certificate expiration and health monitoring
- Detection of SSL configuration vulnerabilities and compliance issues
- Proactive alerting on expiring certificates or security problems to reduce downtime and security risks
- Centralized reporting with historical tracking potential

**Logical Blocks:**

- **1.1 Trigger and Data Input:** Scheduled daily trigger and fetching monitored domain list from Notion
- **1.2 Dual SSL Data Collection:** Perform SSL health analysis via SSH script and external API call to ssl-checker.io
- **1.3 SSL Expiry Filtering and Alerting:** Filter domains with imminent expiry and send Discord alerts for expiry warnings
- **1.4 SSL Scanner Output Processing:** Parse and format SSH scanner results with detailed health assessment and generate comprehensive Discord alerts

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger and Data Input

**Overview:**  
This block initiates the workflow every day at 10:00 AM and retrieves the list of domains to monitor from a Notion database.

**Nodes Involved:**  
- Daily Trigger  
- Fetch domains to check SSL  

**Node Details:**

- **Daily Trigger**  
  - Type: Schedule Trigger  
  - Configuration: Runs daily at 10:00 AM (hour=10)  
  - Input: None (trigger node)  
  - Output: Triggers downstream nodes once daily  
  - Edge Cases: System downtime at trigger time may delay workflow start

- **Fetch domains to check SSL**  
  - Type: Notion (databasePage - getAll)  
  - Configuration: Retrieves all pages from a configured Notion database containing URLs for SSL monitoring  
  - Key Parameters: `databaseId` pointing to the Notion database with domains  
  - Input: Trigger from Daily Trigger  
  - Output: Outputs JSON array of domains to be checked  
  - Edge Cases: API rate limits, Notion API credential failures, data schema mismatch (missing URL column)  

---

#### 1.2 Dual SSL Data Collection

**Overview:**  
This block performs dual SSL checks per domain: an internal SSL health assessment via an SSH-executed Node.js script and an external API call to ssl-checker.io for verification.

**Nodes Involved:**  
- SSH - Analyze system  
- Check SSL  

**Node Details:**

- **SSH - Analyze system**  
  - Type: SSH  
  - Role: Executes a Node.js SSL health assessment script on a remote system  
  - Configuration: Runs command `node /opt/sysadmin-toolkit/scripts/ssl/ssl-health-assessment.js {{ $json.property_domains }} --json`  
  - Authentication: Private key-based SSH authentication  
  - Input: Domains from Notion fetch node  
  - Output: JSON results with detailed SSL health status, grades, vulnerabilities, and recommendations  
  - Edge Cases: SSH connection failure, script errors, malformed JSON output, timeouts, private key misconfiguration  

- **Check SSL**  
  - Type: HTTP Request  
  - Role: Queries external API `ssl-checker.io` for SSL certificate data including expiration and host info  
  - Configuration: URL dynamically constructed by stripping protocol and trailing slash from input domain  
  - Input: Domains from Notion fetch node  
  - Output: JSON with certificate validity dates and days left  
  - Edge Cases: API downtime, HTTP request timeouts, invalid domain formats, rate limiting  

---

#### 1.3 SSL Expiry Filtering and Alerting

**Overview:**  
Filters domains where SSL certificate expiry is within a critical threshold (7 days or less) and sends immediate Discord alerts for these cases.

**Nodes Involved:**  
- Expiry Alert  
- Discord1  

**Node Details:**

- **Expiry Alert**  
  - Type: If (conditional)  
  - Role: Checks if `days_left` from ssl-checker.io API response is less than or equal to 7  
  - Configuration: Numeric condition `days_left <= 7`  
  - Input: Output from Check SSL node  
  - Output: Passes only domains meeting expiry condition  
  - Edge Cases: Missing or malformed `days_left` data, zero or negative values  

- **Discord1**  
  - Type: Discord  
  - Role: Sends a Discord message alerting about imminent SSL expiry  
  - Configuration: Sends message formatted as `SSL Expiry - {{ $json.result.days_left }} days Left - {{ $json.result.host }}`  
  - Targets specific guild and channel via webhook and IDs  
  - Input: Filtered domains from Expiry Alert  
  - Edge Cases: Discord webhook misconfiguration, rate limits, message formatting errors  

---

#### 1.4 SSL Scanner Output Processing

**Overview:**  
Parses detailed SSL health scan results from the SSH node, evaluates alert conditions, formats rich Discord notification content, and sends alerts based on health scores and vulnerabilities.

**Nodes Involved:**  
- Code - Format output  
- Discord  

**Node Details:**

- **Code - Format output**  
  - Type: Code (JavaScript)  
  - Role: Processes SSH node JSON output to:  
    - Detect errors or failures  
    - Extract SSL grades, certificate info, protocol support, vulnerabilities  
    - Determine alert level (critical, high, medium, low, info) based on multiple conditions (expiry, vulnerabilities, protocol deprecation, grade)  
    - Compose detailed Discord embed content including color coding, fields, descriptions, and summaries  
  - Input: Output from SSH - Analyze system node  
  - Output: Structured JSON for Discord notification  
  - Edge Cases: Parsing failures, unexpected JSON structure, runtime exceptions (caught/reported), missing data fields  

- **Discord**  
  - Type: Discord  
  - Role: Sends rich alert messages to Discord channel with detailed SSL health status and recommendations  
  - Configuration: Uses webhook with dynamic content from Code node including title, description, and embed fields  
  - Input: Formatted JSON from Code - Format output  
  - Edge Cases: Discord API errors, webhook invalidation, message throttling  

---

### 3. Summary Table

| Node Name               | Node Type               | Functional Role                               | Input Node(s)                 | Output Node(s)                   | Sticky Note                                                                                                                               |
|-------------------------|-------------------------|-----------------------------------------------|------------------------------|---------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------|
| Daily Trigger           | Schedule Trigger        | Initiates workflow daily at 10:00 AM           | None                         | Fetch domains to check SSL       | ### 🕒 Daily Trigger Runs the workflow every day at 10:00 AM.  Adjust time/frequency via cron or interval.                                 |
| Fetch domains to check SSL | Notion                 | Retrieves monitored domains from Notion        | Daily Trigger                | SSH - Analyze system, Check SSL  | ### 📄 Fetch URLs Reads website URLs from Notion. Ensure a `URL` column exists with domains to monitor.                                    |
| SSH - Analyze system    | SSH                     | Runs internal SSL health assessment script     | Fetch domains to check SSL   | Code - Format output             | ### 🔍 Check SSL health score Use sysadmin-toolkit/scripts/ssl/ssl-health-assessment.js. Returns SSL overall grade, vulnerabilities, actions |
| Check SSL               | HTTP Request            | Calls external API ssl-checker.io for cert info | Fetch domains to check SSL   | Expiry Alert                    | ### 🔍 Check SSL Queries ssl-checker.io API for SSL certificate data. Returns valid_from, valid_till, days_left, host info.               |
| Expiry Alert            | If                      | Filters domains with SSL expiry ≤ 7 days       | Check SSL                   | Discord1                       | ### ⚠️ Expiry Alert Checks if `days_left` ≤ 7 to trigger alerts.                                                                          |
| Discord1                | Discord                 | Sends Discord alerts for imminent expiry       | Expiry Alert                | None                           | ### 📧 Send Discord alerts Sends alerts when there are problems.                                                                          |
| Code - Format output    | Code                    | Parses SSH SSL scan output and formats alerts  | SSH - Analyze system        | Discord                       |                                                                                                                                           |
| Discord                 | Discord                 | Sends detailed Discord alerts with SSL health  | Code - Format output        | None                           | ### 📧 Send Discord alerts Sends alerts whether there are problems.                                                                       |
| Sticky Note             | Sticky Note             | Workflow overview and documentation             | None                        | None                           | ## 🔐 Advanced SSL Health Monitor (full descriptive content on workflow purpose and usage).                                                |
| Sticky Note1            | Sticky Note             | Notes on Daily Trigger                           | None                        | None                           | ### 🕒 Daily Trigger Runs every day at 10:00 AM.                                                                                          |
| Sticky Note2            | Sticky Note             | Notes on Fetch URLs                              | None                        | None                           | ### 📄 Fetch URLs Read URLs from Notion with URL column.                                                                                  |
| Sticky Note3            | Sticky Note             | Notes on Check SSL                               | None                        | None                           | ### 🔍 Check SSL Queries ssl-checker.io API for cert data.                                                                                |
| Sticky Note4 (disabled) | Sticky Note             | Notes on Expiry Alert                            | None                        | None                           | ### ⚠️ Expiry Alert Checks if days_left ≤ 7 to trigger alert emails. (disabled)                                                           |
| Sticky Note5 (disabled) | Sticky Note             | Notes on Discord alerts                          | None                        | None                           | ### 📧 Send Discord alerts Sends alerts on problems. (disabled)                                                                           |
| Sticky Note6 (disabled) | Sticky Note             | Notes on Discord alerts                          | None                        | None                           | ### 📧 Send Discord alerts Sends alerts on problems. (disabled)                                                                           |
| Sticky Note7            | Sticky Note             | Notes on SSL health score                        | None                        | None                           | ### 🔍 Check SSL health score Runs sysadmin-toolkit SSL health assessment script.                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node** named `Daily Trigger`:
   - Set it to trigger every day at 10:00 AM.
   - Use interval trigger with hour = 10 or equivalent cron expression.

2. **Create a Notion node** named `Fetch domains to check SSL`:
   - Configure to use Notion credentials.
   - Set resource to `databasePage` and operation to `getAll`.
   - Provide your Notion database ID containing the monitored domains.
   - Ensure the database has a column named `URL` with domain URLs.
   - Connect `Daily Trigger` output to this node’s input.

3. **Create an SSH node** named `SSH - Analyze system`:
   - Configure SSH credentials with private key authentication to your remote server.
   - Command to execute:  
     `node /opt/sysadmin-toolkit/scripts/ssl/ssl-health-assessment.js {{ $json.property_domains }} --json`  
     (replace `property_domains` with the actual JSON path containing the domain string)
   - Connect output of `Fetch domains to check SSL` to this node.

4. **Create an HTTP Request node** named `Check SSL`:
   - Set to GET request.
   - URL expression:  
     `https://ssl-checker.io/api/v1/check/{{ $json["property_domains"].replace(/^https?:\/\/|\/$/g, "") }}`
   - Connect output of `Fetch domains to check SSL` to this node.

5. **Create an If node** named `Expiry Alert`:
   - Set condition to check if `days_left` (path: `$json.result.days_left`) is less than or equal to 7.
   - Connect output of `Check SSL` to this node.

6. **Create a Discord node** named `Discord1` for expiry alerts:
   - Use Discord webhook credentials.
   - Set message content:  
     `SSL Expiry - {{ $json.result.days_left }} days Left - {{ $json.result.host }}`
   - Configure guild and channel IDs accordingly.
   - Connect `Expiry Alert` true output to this node.

7. **Create a Code node** named `Code - Format output`:
   - Paste the provided JavaScript code that:  
     - Parses SSH node output  
     - Handles errors and alert levels  
     - Constructs Discord embed content  
   - Connect output of `SSH - Analyze system` to this node.

8. **Create a Discord node** named `Discord` for detailed alerts:
   - Use Discord webhook credentials.
   - Set message content with the dynamic fields from `Code - Format output` (e.g., `{{ $json.discord.title }}`, `{{ $json.discord.fullDescription }}`).
   - Configure guild and channel IDs accordingly.
   - Connect output of `Code - Format output` to this node.

9. **Connect nodes to form the workflow**:  
   - `Daily Trigger` → `Fetch domains to check SSL`  
   - `Fetch domains to check SSL` → `SSH - Analyze system` → `Code - Format output` → `Discord`  
   - `Fetch domains to check SSL` → `Check SSL` → `Expiry Alert` → `Discord1`

10. **Configure credentials**:  
    - Setup Notion API credentials with access to the monitoring database.  
    - Setup SSH credentials with private key access to the server running the SSL health assessment script.  
    - Setup Discord webhook credentials with access to your Discord server and alert channel.

11. **Optional**: Adjust alert thresholds, Discord message formatting, and Notion database schema according to your environment.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Context or Link                                                                                      |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This workflow transforms basic SSL expiry checking into a comprehensive SSL security monitoring solution rivaling enterprise-grade tools while remaining open-source and cost-effective. It uses dual verification (internal scanner + ssl-checker.io) for high reliability. Alerts use rich Discord embeds with color coding and detailed recommendations.                                                                                                                                                                                                                                                                                                                                          | Workflow Overview Sticky Note                                                                       |
| Setup instructions emphasize configuring Notion with a `URL` column, Discord webhook for alerts, and SSH access to the scanner server. Thresholds for expiry alerts and SSL grade can be customized.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Setup Instructions Sticky Note                                                                      |
| The SSH node executes a custom Node.js script located at `/opt/sysadmin-toolkit/scripts/ssl/ssl-health-assessment.js` which must be pre-installed on the target server. This script returns JSON with SSL grades, vulnerabilities, and recommendations.                                                                                                                                                                                                                                                                                                                                                                                                                                             | SSH Node Sticky Note                                                                                |
| Discord messages include pre-formatted summaries, detailed fields (certificate info, security status, recommendations), and use color codes to highlight alert severity. Critical alerts ping `@here`.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Code node logic description                                                                         |
| The external API used (`ssl-checker.io`) is a free SSL certificate checker providing expiration and validity details for cross-verification.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | HTTP Request node Sticky Note                                                                       |
| The workflow is designed with scalability in mind to handle hundreds of domains and avoid alert noise by only notifying on actual issues.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | General workflow design notes                                                                       |
| Discord webhook URLs and IDs are pre-configured and should be updated for your specific server, guild, and channel.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Discord nodes configuration notes                                                                  |
| For further customization, vulnerability checks, compliance standards, and alert severity levels can be adjusted by modifying the code node logic.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Customization notes                                                                                |

---

**Disclaimer:**  
The text provided is generated exclusively from an n8n automated workflow export. It complies strictly with current content policies and contains no illegal, offensive, or protected content. All data processed is legal and public.
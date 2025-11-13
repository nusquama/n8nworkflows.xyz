Automated Server Log Cleanup via Email Alerts with SSH - Nginx, Docker, System

https://n8nworkflows.xyz/workflows/automated-server-log-cleanup-via-email-alerts-with-ssh---nginx--docker--system-5406


# Automated Server Log Cleanup via Email Alerts with SSH - Nginx, Docker, System

### 1. Workflow Overview

This workflow automates server disk cleanup triggered by incoming email alerts about disk utilization. When an alert email is received, the workflow extracts the server IP address from the email subject and uses SSH to execute a series of cleanup commands remotely. This process targets cleaning log files, Docker caches, system logs, and package caches on the affected server to free disk space.

The workflow is logically divided into the following blocks:

- **1.1 Email Alert Reception:** Monitor and filter incoming emails for disk utilization alerts.
- **1.2 Data Extraction:** Parse the email to extract the target server IP address.
- **1.3 SSH Preparation:** Configure the necessary credentials and variables for remote SSH execution.
- **1.4 Remote Cleanup Execution:** Run a scripted set of commands on the server via SSH to purge logs and perform system cleanup tasks.

---

### 2. Block-by-Block Analysis

#### 2.1 Email Alert Reception

- **Overview:**  
  This block listens for new, unread emails with subjects containing the keyword "disk" to identify disk utilization alerts.

- **Nodes Involved:**  
  - Check Disk Alert Emails

- **Node Details:**  

  **Check Disk Alert Emails**  
  - Type: IMAP Email Read  
  - Role: Fetch unread emails with specific subject criteria.  
  - Configuration: Uses IMAP credentials; filters for unseen emails with subject containing "disk".  
  - Key Expressions: Custom email config `["UNSEEN", ["SUBJECT", "disk"]]`.  
  - Input: Triggered automatically (start node).  
  - Output: Passes matched email data to next node.  
  - Potential Failures: IMAP connection failures, authentication issues, no matching emails (which results in no output).  
  - Sticky Note: "Triggers when an alert email about disk usage is received."

#### 2.2 Data Extraction

- **Overview:**  
  Extracts the IP address of the affected server from the email subject line if it contains "disk-utilization" or "disk utilization".

- **Nodes Involved:**  
  - Extract Server IP from Email

- **Node Details:**  

  **Extract Server IP from Email**  
  - Type: Code (JavaScript)  
  - Role: Parse email subject string to extract server IP using regex.  
  - Configuration:  
    - Checks if subject contains keywords "disk-utilization" or "disk utilization".  
    - Applies regex `(\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}):9100` to extract IP preceding port 9100.  
    - Returns empty output if no match, effectively stopping workflow.  
  - Key Expressions: Uses `$input.first().json.subject`.  
  - Input: Output from Check Disk Alert Emails.  
  - Output: JSON object with `server_ip` property if matched.  
  - Potential Failures:  
    - No subject or malformed subject line leads to empty output (stops workflow).  
    - Regex mismatch leads to empty IP field.  
    - Expression errors if input is missing.  
  - Sticky Note: "Parses the email body to extract the target server's IP address."

#### 2.3 SSH Preparation

- **Overview:**  
  Sets SSH connection variables including password, username, and the extracted server IP address for downstream SSH node usage.

- **Nodes Involved:**  
  - Prepare SSH Variables

- **Node Details:**  

  **Prepare SSH Variables**  
  - Type: Set  
  - Role: Define and assign fixed and dynamic variables for SSH connection.  
  - Configuration:  
    - `pwd`: hardcoded password string (e.g., "password").  
    - `server_user`: hardcoded username string (e.g., "user").  
    - `server_ip`: dynamically set from previous node's `server_ip`.  
  - Key Expressions: `={{ $input.first().json.server_ip }}` for dynamic IP.  
  - Input: Output from Extract Server IP from Email.  
  - Output: Variables passed to SSH execution node.  
  - Potential Failures: Hardcoded credentials pose security issues; missing IP leads to empty variable.  
  - Sticky Note: "Manually set or map credentials, paths, or other config values."

#### 2.4 Remote Cleanup Execution

- **Overview:**  
  Executes a comprehensive set of system cleanup commands remotely over SSH for logs, Docker cache, and package management.

- **Nodes Involved:**  
  - Run Log Cleanup Commands via SSH

- **Node Details:**  

  **Run Log Cleanup Commands via SSH**  
  - Type: SSH  
  - Role: Connects to remote server and runs cleanup shell commands.  
  - Configuration:  
    - Command string constructs a pipeline:  
      - Uses echo to send password for sudo.  
      - Switches user to `jenkins`.  
      - SSH into target server with provided user and IP.  
      - Runs commands: remove compressed and rotated logs, clear nginx and apache logs, vacuum systemd journal to 200MB, clean apt cache, prune docker builder, show docker system disk usage, autoremove unused packages.  
  - Credentials: Uses SSH password credential for Jenkins Server.  
  - Key Expressions: Multi-level interpolation of password, user, and IP variables.  
  - Input: Variables from Prepare SSH Variables node.  
  - Output: Command execution results (stdout/stderr).  
  - Potential Failures:  
    - SSH connection failures (network, auth, firewall).  
    - Incorrect password or username leads to auth failure.  
    - Command execution errors (permission denied, command not found).  
    - Long command could timeout or partial execution.  
  - Sticky Note: "Executes cleanup commands (Nginx, PM2, Docker, system logs)."

---

### 3. Summary Table

| Node Name                      | Node Type            | Functional Role                   | Input Node(s)                | Output Node(s)                     | Sticky Note                                                                 |
|-------------------------------|----------------------|---------------------------------|-----------------------------|----------------------------------|-----------------------------------------------------------------------------|
| Check Disk Alert Emails        | IMAP Email Read      | Detect disk usage alert emails   | (start)                     | Extract Server IP from Email      | Triggers when an alert email about disk usage is received.                  |
| Extract Server IP from Email   | Code (JavaScript)    | Parse server IP from email       | Check Disk Alert Emails      | Prepare SSH Variables             | Parses the email body to extract the target server's IP address.            |
| Prepare SSH Variables          | Set                  | Configure SSH connection vars    | Extract Server IP from Email | Run Log Cleanup Commands via SSH  | Manually set or map credentials, paths, or other config values.             |
| Run Log Cleanup Commands via SSH | SSH               | Execute remote cleanup commands  | Prepare SSH Variables        | (end)                            | Executes cleanup commands (Nginx, PM2, Docker, system logs).                |
| Sticky Note                   | Sticky Note           | Documentation                   |                             |                                  |                                                                             |
| Sticky Note1                  | Sticky Note           | Documentation                   |                             |                                  |                                                                             |
| Sticky Note2                  | Sticky Note           | Documentation                   |                             |                                  |                                                                             |
| Sticky Note3                  | Sticky Note           | Documentation                   |                             |                                  |                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Check Disk Alert Emails" Node**  
   - Type: EmailReadImap  
   - Configure IMAP credentials (e.g., your email account).  
   - Set options to filter unread emails with subject containing "disk": `["UNSEEN", ["SUBJECT", "disk"]]`.  
   - Position at start of workflow.

2. **Create "Extract Server IP from Email" Node**  
   - Type: Code (JavaScript)  
   - Connect input from "Check Disk Alert Emails".  
   - Use this script:  
     ```javascript
     const subject = $input.first().json.subject;

     if (subject && (subject.toLowerCase().includes("disk-utilization") || subject.toLowerCase().includes("disk utilization"))) {
       const ipMatch = subject.match(/(\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}):9100/);
       const server_ip = ipMatch ? ipMatch[1] : '';
       return [{ json: { server_ip } }];
     } else {
       return [];
     }
     ```
   - This node filters relevant emails and extracts IP.

3. **Create "Prepare SSH Variables" Node**  
   - Type: Set  
   - Connect input from "Extract Server IP from Email".  
   - Add three variables:  
     - `pwd`: string, your password for sudo.  
     - `server_user`: string, your SSH username.  
     - `server_ip`: expression set to `{{$input.first().json.server_ip}}`.  
   - These variables prepare SSH connection details.

4. **Create "Run Log Cleanup Commands via SSH" Node**  
   - Type: SSH  
   - Connect input from "Prepare SSH Variables".  
   - Configure SSH credential (password or key-based) for the Jenkins server or relevant user.  
   - Set command as:  
     ```bash
     echo '{{ $json.pwd }}' | sudo -S su - jenkins -c "ssh {{ $json.server_user }}@{{ $json.server_ip }} 'sudo rm -rf /var/log/*.gz && sudo rm -rf /var/log/*.1 && sudo rm -rf /var/log/nginx/*.log && sudo rm -rf /var/log/apache2/*.log && sudo journalctl --vacuum-size=200M && sudo apt-get clean && sudo apt-get autoclean && docker builder prune -af && docker system df && sudo apt-get autoremove -y'"
     ```
   - This command runs log cleanup and system maintenance on the remote server.

5. **Add Sticky Notes (Optional for Documentation)**  
   - Add sticky notes near each node with the respective descriptions for clarity.

6. **Set Workflow Activation**  
   - Test workflow with sample alert emails.  
   - Enable execution triggers to run periodically or on-demand.

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                         |
|------------------------------------------------------------------------------------------------------|--------------------------------------------------------|
| Hardcoded passwords in workflows are a security risk; consider using n8n credentials or environment variables for secrets. | Security best practices                                |
| SSH commands require proper permissions on remote servers; ensure user keys and sudoers configuration allow commands. | Server administration                                  |
| Regex for IP extraction assumes format `xxx.xxx.xxx.xxx:9100` in email subject; adjust regex if email format changes. | Email parsing specifics                                 |
| Docker cleanup commands prune builder caches to free disk space, useful in CI/CD environments.       | Docker maintenance                                     |
| System logs vacuum and apt-get housekeeping commands help maintain server health and disk availability.| Linux system maintenance                               |

---

**Disclaimer:**  
The provided text comes exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.
Enrich IP addresses with country attribution using IPinfo and Slack alerts

https://n8nworkflows.xyz/workflows/enrich-ip-addresses-with-country-attribution-using-ipinfo-and-slack-alerts-13239


# Enrich IP addresses with country attribution using IPinfo and Slack alerts

## 1. Workflow Overview

**Purpose:** This workflow receives an IP address via an HTTP webhook, validates it (IPv4), ignores private/internal ranges, enriches public IPs using IPinfo (geo + ASN/org), normalizes the enrichment into a consistent schema with a simple country-based risk label, sends a Slack notification, and returns a structured JSON response to the caller.

**Target use cases:**
- SOC/security automation for quick IP triage (geo attribution + ISP/ASN context)
- Lightweight enrichment endpoint for other tools (SIEM/SOAR scripts, Slack slash commands, internal APIs)

### Logical blocks
**1.1 Input Reception & Validation**
- Webhook receives the request, Code validates IPv4 shape/range, IF routes valid vs invalid.

**1.2 Private/Internal IP Handling**
- Code checks RFC1918/loopback ranges, IF routes private vs public; private gets an “ignored” response.

**1.3 Public IP Enrichment (IPinfo)**
- HTTP request to `ipinfo.io` for public IPs.

**1.4 Normalization & Severity**
- Code normalizes fields (country/region/city/org/asn) and tags severity based on country list.

**1.5 Notification & Output**
- Slack message posted to a channel, then Respond to Webhook returns the final JSON enrichment to the original caller.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception & Validation

**Overview:** Accepts inbound POST requests, extracts an IP from the request body, validates it as IPv4, and short-circuits with an error response if invalid.

**Nodes involved:**
- Weebhook - Receive IP Input
- Validate IP Address
- If IP is Valid
- Respond Invalid IP Address

#### Node: **Weebhook - Receive IP Input** (Webhook)
- **Type / role:** `n8n-nodes-base.webhook` — entry point HTTP listener.
- **Key configuration:**
  - **Method:** `POST`
  - **Path:** `/ip-enrichment`
  - **Response mode:** `responseNode` (a downstream Respond to Webhook node must answer)
- **Expected input shape:**
  - The next Code node reads: `{{$input.first().json.body.text}}`
  - So the workflow expects a JSON body with a `text` field, e.g.:
    - `{ "text": "8.8.8.8" }`
- **Connections:**
  - Output → **Validate IP Address**
- **Failure/edge cases:**
  - If caller sends non-JSON or a body without `text`, validation will fail and return invalid response.
  - If no Respond node executes (misrouting), the webhook call may hang until timeout.

#### Node: **Validate IP Address** (Code)
- **Type / role:** `n8n-nodes-base.code` — parses and validates IPv4.
- **Configuration choices:**
  - Reads IP from `body.text`.
  - Validates basic IPv4 regex and octet numeric range (0–255).
  - If missing IP: returns `{ valid:false, reason:"IP missing" }` (note: reason is not used downstream).
- **Key variables/expressions:**
  - `const ip = $input.first().json.body.text;`
  - Regex: `/^(\d{1,3}\.){3}\d{1,3}$/`
- **Outputs:**
  - `{ ip, valid }` (and sometimes `{ valid:false, reason:"IP missing" }` but without `ip`)
- **Connections:**
  - Output → **If IP is Valid**
- **Failure/edge cases:**
  - If `body` is undefined (e.g., different content type), `body.text` access may throw in some runtime contexts. In n8n Code node, `json.body` is usually present for webhook JSON; but if not, this can fail.
  - Only IPv4 supported (no IPv6).

#### Node: **If IP is Valid** (IF)
- **Type / role:** `n8n-nodes-base.if` — branching.
- **Condition:**
  - Checks boolean true: `{{ $json.valid }} is true`
- **Connections:**
  - **True** → **Check Private / Internal IP**
  - **False** → **Respond Invalid IP Address**
- **Failure/edge cases:**
  - If upstream output lacks `valid` (unexpected), strict type validation may treat it as false → invalid response.

#### Node: **Respond Invalid IP Address** (Respond to Webhook)
- **Type / role:** `n8n-nodes-base.respondToWebhook` — returns immediate HTTP response.
- **Response:**
  - JSON:
    ```json
    { "status": "error", "message": "Invalid IP address." }
    ```
- **Connections:** none (terminal for invalid branch)
- **Failure/edge cases:**
  - If you want to return the specific reason (“IP missing”), you’d need to include `$json.reason` here.

---

### 2.2 Private/Internal IP Handling

**Overview:** Detects private/internal IPv4 ranges and stops enrichment for those IPs, returning an “ignored” status instead of calling external services.

**Nodes involved:**
- Check Private / Internal IP
- If IP is Private
- Respond Private IP Ignored

#### Node: **Check Private / Internal IP** (Code)
- **Type / role:** `n8n-nodes-base.code` — classifies IP as private/internal.
- **Configuration choices:**
  - Uses regex checks for:
    - `10.0.0.0/8`
    - `192.168.0.0/16`
    - `172.16.0.0/12`
    - `127.0.0.0/8` (loopback)
- **Key variables:**
  - `const ip = $input.first().json.ip;`
  - `isPrivate = privateRanges.some(r => r.test(ip));`
- **Outputs:** `{ ip, isPrivate }`
- **Connections:**
  - Output → **If IP is Private**
- **Failure/edge cases:**
  - Does not include other non-public ranges (e.g., `169.254.0.0/16` link-local, `100.64.0.0/10` CGNAT, multicast/reserved ranges). If needed, extend regex list.

#### Node: **If IP is Private** (IF)
- **Type / role:** branching.
- **Condition:** `{{ $json.isPrivate }} is true`
- **Connections:**
  - **True** → **Respond Private IP Ignored**
  - **False** → **Enrich IP (Geo & ASN Lookup)**
- **Failure/edge cases:**
  - If `isPrivate` missing/undefined, strict boolean check evaluates false → will proceed to enrichment.

#### Node: **Respond Private IP Ignored** (Respond to Webhook)
- **Type / role:** terminates webhook with an “ignored” result.
- **Response body (JSON):**
  - Includes IP via expression: `{{ $json.ip }}`
  - Example:
    ```json
    { "status":"ignored", "message":"Private or internal IP address", "ip":"192.168.1.10" }
    ```
- **Connections:** none
- **Failure/edge cases:**
  - Works only if upstream provides `ip`.

---

### 2.3 Public IP Enrichment (IPinfo)

**Overview:** For public IPs, fetches geo and organization/ASN information from IPinfo.

**Nodes involved:**
- Enrich IP (Geo & ASN Lookup)

#### Node: **Enrich IP (Geo & ASN Lookup)** (HTTP Request)
- **Type / role:** `n8n-nodes-base.httpRequest` — calls external enrichment API.
- **Configuration choices:**
  - **URL:** `https://ipinfo.io/{{ $json.ip }}/json`
  - **Response format:** JSON
- **Inputs:** `{ ip, isPrivate:false }`
- **Outputs:** Whatever IPinfo returns (commonly includes `ip`, `country`, `region`, `city`, `org`, etc.)
- **Connections:**
  - Output → **Normalize Enrichment**
- **Version-specific requirements:**
  - Node typeVersion `4.3` (modern HTTP Request node behavior; UI differs from older versions).
- **Failure/edge cases:**
  - **Rate limits / 429:** IPinfo free tier limits can cause failures.
  - **Auth:** This workflow does not pass an IPinfo token. Without a token, responses may be rate-limited or less reliable. Consider adding `?token=...`.
  - **Network/DNS/timeouts:** external call can fail; add retries or error handling if needed.
  - **Unexpected fields:** if IPinfo omits `org` or `country`, normalization must handle it (it mostly does).

---

### 2.4 Normalization & Severity

**Overview:** Maps IPinfo fields to a consistent output schema, derives ASN from org string, and assigns a severity label based on a hardcoded high-risk country list.

**Nodes involved:**
- Normalize Enrichment

#### Node: **Normalize Enrichment** (Code)
- **Type / role:** `n8n-nodes-base.code` — transforms IPinfo output.
- **Configuration choices:**
  - High-risk countries: `["RU","CN","IR","KP"]`
  - Severity:
    - High risk → `"🚨 HIGH RISK"`
    - Otherwise → `"🟢 Normal"`
  - Normalized fields with defaults:
    - `country/region/city/isp` default to `"Unknown"`
    - `asn` derived from `org.split(" ")[0]` else `"N/A"`
- **Key variables:**
  - `const data = $json;`
  - `data.org?.split(" ")[0]` (optional chaining)
- **Outputs:**
  - `{ ip, country, region, city, isp, asn, severity }`
- **Connections:**
  - Output → **Slack Alert: IP Enrichment Result**
- **Failure/edge cases:**
  - `asn` derivation assumes `org` begins with something like `AS15169 ...`. If IPinfo returns a different org format, ASN may be incorrect.
  - Severity is computed but not currently included in Slack message nor webhook response (it is present in node output but not used downstream). If you want it visible, add it to Slack text and response body.

---

### 2.5 Notification & Output

**Overview:** Posts the normalized enrichment to Slack and returns a JSON response to the webhook caller.

**Nodes involved:**
- Slack Alert: IP Enrichment Result
- Respond IP Enrichment Result

#### Node: **Slack Alert: IP Enrichment Result** (Slack)
- **Type / role:** `n8n-nodes-base.slack` — sends a Slack message.
- **Configuration choices:**
  - **Operation:** Post message to **channel** (selected from list)
  - **Channel:** `all-team-sawi` (ID `C0A252GLT70`)
  - **Message text:** templated with:
    - `{{$json.ip}}`, `{{$json.country}}`, `{{$json.region}}`, `{{$json.city}}`, `{{$json.isp}}`, `{{$json.asn}}`
  - **Other option:** `includeLinkToWorkflow: false`
- **Credentials:**
  - Slack API credential named **“Slack account 2”**
- **Connections:**
  - Output → **Respond IP Enrichment Result**
- **Failure/edge cases:**
  - Slack auth failures (revoked token, missing scopes).
  - Channel ID invalid or bot not in channel.
  - Message formatting: severity exists upstream but isn’t included; add `• *Severity:* {{$json.severity}}` if desired.

#### Node: **Respond IP Enrichment Result** (Respond to Webhook)
- **Type / role:** `n8n-nodes-base.respondToWebhook` — final response to the initial webhook request.
- **Response body (JSON):**
  - Uses expressions for `ip/country/region/city/isp/asn`.
  - Does **not** include `severity` (available but unused).
- **Connections:** none (terminal success path)
- **Failure/edge cases:**
  - If Slack node fails and stops execution, this response may never be sent (depending on error handling). If you want the webhook to respond even when Slack fails, add error handling (e.g., Slack node “Continue On Fail” or separate branch).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Weebhook - Receive IP Input | Webhook | Receive IP via HTTP POST and wait for response node | — | Validate IP Address | ## 1. Input & Validation |
| Validate IP Address | Code | Extract IP from request body and validate IPv4 | Weebhook - Receive IP Input | If IP is Valid | ## 1. Input & Validation |
| If IP is Valid | IF | Route valid vs invalid IP | Validate IP Address | Check Private / Internal IP; Respond Invalid IP Address | ## 1. Input & Validation |
| Respond Invalid IP Address | Respond to Webhook | Return error JSON for invalid input | If IP is Valid (false) | — | ## 1. Input & Validation |
| Check Private / Internal IP | Code | Detect RFC1918/loopback private ranges | If IP is Valid (true) | If IP is Private | ## 2. Private IP Handling |
| If IP is Private | IF | Route private vs public IP | Check Private / Internal IP | Respond Private IP Ignored; Enrich IP (Geo & ASN Lookup) | ## 2. Private IP Handling |
| Respond Private IP Ignored | Respond to Webhook | Return “ignored” JSON for private/internal IPs | If IP is Private (true) | — | ## 2. Private IP Handling |
| Enrich IP (Geo & ASN Lookup) | HTTP Request | Call IPinfo for geo/org enrichment | If IP is Private (false) | Normalize Enrichment | ## IP Enrichment |
| Normalize Enrichment | Code | Normalize fields and compute severity | Enrich IP (Geo & ASN Lookup) | Slack Alert: IP Enrichment Result | ## Normalization & Severity |
| Slack Alert: IP Enrichment Result | Slack | Send enrichment results to Slack channel | Normalize Enrichment | Respond IP Enrichment Result | ## Output & Notification |
| Respond IP Enrichment Result | Respond to Webhook | Return final enrichment JSON to caller | Slack Alert: IP Enrichment Result | — | ## Output & Notification |
| Sticky Note | Sticky Note | Comment block header | — | — | ## 1. Input & Validation |
| Sticky Note1 | Sticky Note | Comment block header | — | — | ## 2. Private IP Handling |
| Sticky Note2 | Sticky Note | Comment block header | — | — | ## IP Enrichment |
| Sticky Note3 | Sticky Note | Comment block header | — | — | ## Normalization & Severity |
| Sticky Note4 | Sticky Note | Comment block header | — | — | ## Output & Notification |
| Sticky Note5 | Sticky Note | General workflow description and setup/customization notes | — | — | ## I'm a note  / IP Enrichment & Attribution is a lightweight cybersecurity automation designed to enrich IP addresses with geographic and network intelligence. / How it works (1–7) / Setup steps (1–4) / Customization (1–3) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name: **IP Enrichment & Country Attribution**
   - (Optional) Add tags: `ip-enrichment`, `geoip`, `threat-intelligence`.

2. **Add Webhook node**
   - Node type: **Webhook**
   - Name: **Weebhook - Receive IP Input**
   - HTTP Method: **POST**
   - Path: **/ip-enrichment**
   - Response Mode: **Using “Respond to Webhook” node** (responseNode)

3. **Add Code node: Validate IPv4**
   - Node type: **Code**
   - Name: **Validate IP Address**
   - Paste logic that:
     - Reads `ip` from `{{$input.first().json.body.text}}`
     - Validates IPv4 regex and octets 0–255
     - Outputs `{ ip, valid }` (and optional reason on missing)
   - Connect: **Webhook → Validate IP Address**

4. **Add IF node: valid check**
   - Node type: **IF**
   - Name: **If IP is Valid**
   - Condition (Boolean):
     - Left value: `{{ $json.valid }}`
     - Operation: **is true**
   - Connect: **Validate IP Address → If IP is Valid**

5. **Add Respond node for invalid IP**
   - Node type: **Respond to Webhook**
   - Name: **Respond Invalid IP Address**
   - Respond with: **JSON**
   - Body:
     ```json
     { "status": "error", "message": "Invalid IP address." }
     ```
   - Connect: **If IP is Valid (false) → Respond Invalid IP Address**

6. **Add Code node: private/internal detection**
   - Node type: **Code**
   - Name: **Check Private / Internal IP**
   - Logic:
     - Read `ip` from `{{$input.first().json.ip}}`
     - Check patterns for `10.*`, `192.168.*`, `172.16–31.*`, `127.*`
     - Output `{ ip, isPrivate }`
   - Connect: **If IP is Valid (true) → Check Private / Internal IP**

7. **Add IF node: private check**
   - Node type: **IF**
   - Name: **If IP is Private**
   - Condition:
     - Left value: `{{ $json.isPrivate }}`
     - Operation: **is true**
   - Connect: **Check Private / Internal IP → If IP is Private**

8. **Add Respond node for private IP**
   - Node type: **Respond to Webhook**
   - Name: **Respond Private IP Ignored**
   - Respond with: **JSON**
   - Body:
     ```json
     {
       "status": "ignored",
       "message": "Private or internal IP address",
       "ip": "{{ $json.ip }}"
     }
     ```
   - Connect: **If IP is Private (true) → Respond Private IP Ignored**

9. **Add HTTP Request node for IPinfo**
   - Node type: **HTTP Request**
   - Name: **Enrich IP (Geo & ASN Lookup)**
   - Method: **GET**
   - URL: `https://ipinfo.io/{{ $json.ip }}/json`
   - Response: **JSON**
   - (Recommended) If using a token, change URL to:
     - `https://ipinfo.io/{{ $json.ip }}/json?token={{ $env.IPINFO_TOKEN }}` (or use n8n credential/parameter)
   - Connect: **If IP is Private (false) → Enrich IP (Geo & ASN Lookup)**

10. **Add Code node: normalize + severity**
    - Node type: **Code**
    - Name: **Normalize Enrichment**
    - Logic:
      - Read IPinfo response from `$json`
      - Map to `{ ip, country, region, city, isp, asn, severity }`
      - Severity based on country list `["RU","CN","IR","KP"]`
    - Connect: **Enrich IP (Geo & ASN Lookup) → Normalize Enrichment**

11. **Add Slack node**
    - Node type: **Slack**
    - Name: **Slack Alert: IP Enrichment Result**
    - Operation: **Post message**
    - Target: **Channel**
    - Choose channel (e.g., `all-team-sawi`)
    - Message text (example):
      - Include: IP, country, region, city, ISP/org, ASN
      - (Optional) add severity line: `• *Severity:* {{$json.severity}}`
    - **Credentials setup:**
      - Create/choose **Slack API** credentials (“Slack account 2” equivalent)
      - Ensure bot/token has permission to post to the channel (and is added to the channel if required)
    - Connect: **Normalize Enrichment → Slack Alert: IP Enrichment Result**

12. **Add final Respond to Webhook**
    - Node type: **Respond to Webhook**
    - Name: **Respond IP Enrichment Result**
    - Respond with: **JSON**
    - Body:
      ```json
      {
        "status": "success",
        "ip": "{{ $json.ip }}",
        "country": "{{ $json.country }}",
        "region": "{{ $json.region }}",
        "city": "{{ $json.city }}",
        "isp": "{{ $json.isp }}",
        "asn": "{{ $json.asn }}"
      }
      ```
      - (Optional) include `"severity": "{{ $json.severity }}"`
    - Connect: **Slack Alert: IP Enrichment Result → Respond IP Enrichment Result**

13. **(Optional) Add sticky notes for clarity**
    - Add notes labeling the blocks:
      - “## 1. Input & Validation”
      - “## 2. Private IP Handling”
      - “## IP Enrichment”
      - “## Normalization & Severity”
      - “## Output & Notification”
      - And the longer descriptive note content from the workflow.

14. **Activate workflow**
    - Test with a POST request to the webhook test/production URL:
      - JSON body: `{ "text": "8.8.8.8" }`

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| IP Enrichment & Attribution is a lightweight cybersecurity automation designed to enrich IP addresses with geographic and network intelligence. It helps security teams, SOC analysts, and automation builders quickly understand where an IP comes from, who owns it, and whether it should be treated as potentially risky. | Sticky note (general description) |
| How it works: (1) Receives an IP via webhook (API or Slack). (2) Validates format and rejects invalid. (3) Checks private/internal ranges. (4) Ignores private IPs with clear response. (5) Enriches public IPs using open-source IP intelligence service. (6) Normalizes country/ISP/ASN and applies severity label. (7) Sends results to Slack and returns structured JSON response. | Sticky note (process summary) |
| Setup steps: (1) Import workflow (2) Activate (3) Configure Slack credentials + target channel (4) Send POST request with IP to webhook URL | Sticky note (setup) |
| Customization ideas: add reputation scores; improve country risk logic; SIEM integrations | Sticky note (customization) |
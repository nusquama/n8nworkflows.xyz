Monitor high-value USDT transfers on Ethereum with Airtable and Slack alerts

https://n8nworkflows.xyz/workflows/monitor-high-value-usdt-transfers-on-ethereum-with-airtable-and-slack-alerts-12026


# Monitor high-value USDT transfers on Ethereum with Airtable and Slack alerts

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title (given):** Monitor high-value USDT transfers on Ethereum with Airtable and Slack alerts  
**Workflow name (in JSON):** Smart Contract Event Monitor (Web3)

**Purpose:**  
This workflow checks the Ethereum mainnet once per day, fetches all **USDT (Tether) ERC-20 Transfer events** from the **latest block**, extracts structured transaction details (from/to/value/tx hash), filters for “high-value” transfers above a configured threshold, stores matching transfers in **Airtable**, generates a compact summary, and sends a single **Slack alert**.

**Target use cases:**
- Daily monitoring of large USDT movements for ops/risk/market-intel teams
- Building a searchable log of large transfers in Airtable
- Sending one consolidated Slack notification rather than per-transfer spam

### 1.1 Scheduled Trigger (Input Reception)
Runs automatically at a fixed daily time.

### 1.2 Blockchain Query (Ethereum / Alchemy JSON-RPC)
Fetches latest block number, then pulls USDT `Transfer` logs from that same block.

### 1.3 Log Parsing + High-Value Filtering
Parses log topics/data into addresses and numeric values, then filters by threshold.

### 1.4 Persistence + Notification
Writes high-value transfers to Airtable, summarizes results, and posts to Slack.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduled Trigger (Daily Execution)
**Overview:** Triggers the workflow every day at a specific hour so monitoring happens automatically without manual runs.  
**Nodes involved:** `Daily Check`

#### Node: Daily Check
- **Type / role:** Schedule Trigger (`n8n-nodes-base.scheduleTrigger`) — entry point.
- **Configuration (interpreted):**
  - Runs **daily at 10:00** (server time zone as configured in n8n).
- **Key expressions/variables:** None.
- **Connections:**
  - **Output →** `Get latest blocknumber`
- **Version-specific notes:** TypeVersion `1.2` (standard schedule trigger behavior).
- **Edge cases / failures:**
  - Time zone misunderstandings (n8n instance timezone vs expected business timezone)
  - Workflow inactive (`active: false` in JSON) → no runs until activated

---

### Block 2 — Blockchain Query (Get latest block + USDT Transfer logs)
**Overview:** Uses Alchemy’s Ethereum JSON-RPC endpoint to retrieve the latest block number, then fetches all USDT `Transfer` event logs in that block.  
**Nodes involved:** `Get latest blocknumber`, `Fetch USDT transaction logs`

#### Node: Get latest blocknumber
- **Type / role:** HTTP Request (`n8n-nodes-base.httpRequest`) — JSON-RPC call `eth_blockNumber`.
- **Configuration (interpreted):**
  - **POST** to `https://eth-mainnet.g.alchemy.com/v2/YOUR_API_KEY`
  - Sends JSON body:
    - `method: "eth_blockNumber"`
    - returns latest block number as hex string in `result` (e.g., `"0x1234..."`)
  - Sets `Content-Type: application/json`
- **Key expressions/variables:** None (static body).
- **Connections:**
  - **Input ←** `Daily Check`
  - **Output →** `Fetch USDT transaction logs`
- **Version-specific notes:** TypeVersion `4.3` (newer HTTP node variants; supports `specifyBody: json`).
- **Edge cases / failures:**
  - Invalid/placeholder Alchemy URL (`YOUR_API_KEY`) → 401/403 or request failure
  - Rate limiting / 429 from Alchemy
  - Network timeouts
  - If response shape changes or returns error JSON-RPC payload, downstream expressions expecting `$json.result` will fail

#### Node: Fetch USDT transaction logs
- **Type / role:** HTTP Request (`n8n-nodes-base.httpRequest`) — JSON-RPC call `eth_getLogs`.
- **Configuration (interpreted):**
  - **POST** to same Alchemy endpoint
  - Raw JSON body (string) that queries logs with:
    - `fromBlock: "{{ $json.result }}"`
    - `toBlock: "{{ $json.result }}"`
    - `address: "0xdAC17F958D2ee523a2206206994597C13D831ec7"` (USDT contract)
    - `topics[0]` = ERC-20 `Transfer(address,address,uint256)` topic:
      - `0xddf252ad...523b3ef`
  - Header: `Content-Type: application/json`
- **Key expressions/variables:**
  - Uses the previous node’s `$json.result` (hex block number) for both `fromBlock` and `toBlock` (single-block scan).
- **Connections:**
  - **Input ←** `Get latest blocknumber`
  - **Output →** `Extract transactions details`
- **Version-specific notes:** TypeVersion `4.3`.
- **Edge cases / failures:**
  - If `$json.result` is missing or not hex, JSON-RPC will error
  - Latest block may have **zero USDT transfers** → result array empty (handled later)
  - Large blocks can return many logs; still generally manageable for one block, but can be heavy on peak activity
  - If Alchemy returns a JSON-RPC error object, `$json.result` will be undefined

---

### Block 3 — Parsing Logs + Filter High Value
**Overview:** Converts the raw `eth_getLogs` entries into structured items and filters to keep only transfers above the configured numeric threshold.  
**Nodes involved:** `Extract transactions details`, `Filter high value transaction`

#### Node: Extract transactions details
- **Type / role:** Code (`n8n-nodes-base.code`) — transforms log array into multiple n8n items.
- **Configuration (interpreted):**
  - Reads `const allLogs = $json.result || []`
  - For each log:
    - Validates `topics` length >= 3 (otherwise skips)
    - Extracts:
      - `from` = last 40 hex chars of `topics[1]` prefixed with `0x`
      - `to` = last 40 hex chars of `topics[2]` prefixed with `0x`
      - `value` = `BigInt(log.data).toString()` (USDT uses 6 decimals; this is raw integer units)
      - `blockNumber`, `transactionHash`, and contract `address`
  - Returns **one item per log**
- **Key expressions/variables:**
  - Uses BigInt conversion: `BigInt(hex)` assumes `hex` is a valid `"0x..."` string.
- **Connections:**
  - **Input ←** `Fetch USDT transaction logs`
  - **Output →** `Filter high value transaction`
- **Version-specific notes:** TypeVersion `2` (Code node using JS).
- **Edge cases / failures:**
  - If a log’s `data` is not a proper hex string, `BigInt()` will throw and fail the node
  - Address extraction assumes indexed topics are 32-byte padded (standard ERC-20) — correct for Transfer events
  - `value` is output as a **string**, not a number (important for the IF node)
  - USDT has **6 decimals**; “human USDT” = `value / 1_000_000`

#### Node: Filter high value transaction
- **Type / role:** IF (`n8n-nodes-base.if`) — keeps only items where `value` exceeds threshold.
- **Configuration (interpreted):**
  - Condition: `{{ $json.value }} > 1000000000`
  - “Loose” type validation enabled.
- **Meaning of threshold:**
  - Since USDT has 6 decimals, `1,000,000,000` (raw units) = **1,000 USDT**.
- **Connections:**
  - **Input ←** `Extract transactions details`
  - **True output →** `Save transaction`
  - **False output →** (not connected; items below threshold are dropped)
- **Version-specific notes:** TypeVersion `2.2`.
- **Edge cases / failures:**
  - `value` is a string; IF uses number comparison with loose validation. Extremely large values may exceed JS safe integer if coerced.
  - If coercion fails (non-numeric string), condition may behave unexpectedly.
  - No downstream path for “false” means “no high value transfers” results in **no Airtable save and no Slack message** (unless at least one passes filter).

---

### Block 4 — Store Results + Summarize + Slack Alert
**Overview:** Saves each high-value transfer to Airtable, then generates a single summary message and posts it to Slack.  
**Nodes involved:** `Save transaction`, `Genrate transction summary`, `Send transaction alert`

#### Node: Save transaction
- **Type / role:** Airtable (`n8n-nodes-base.airtable`) — persists each high-value transfer as a record.
- **Configuration (interpreted):**
  - **Operation:** Create record
  - **Base:** `n8n Demo` (ID: `appF2iYPgVqqyXDC1`)
  - **Table:** `Blockchain` (ID: `tblMe6cVaetBcDN11`)
  - Maps fields from the parsed log item:
    - `Contract` ← `{{$json.contract}}`
    - `From Address` ← `{{$json.from}}`
    - `To Address` ← `{{$json.to}}`
    - `Value` ← `{{$json.value}}`
    - `Block NUmber` ← `{{$json.blockNumber}}` (note the field name’s capitalization/typo)
    - `txHash` ← `{{$json.txHash}}`
  - Conversion options disabled (keeps provided types as-is)
- **Credentials:**
  - Uses an Airtable personal access token credential (`airtableTokenApi`).
- **Connections:**
  - **Input ←** `Filter high value transaction` (true branch)
  - **Output →** `Genrate transction summary`
- **Version-specific notes:** TypeVersion `2.1`.
- **Edge cases / failures:**
  - Auth failures (invalid token / expired permissions)
  - Base/table/field mismatch:
    - Airtable field names must match exactly (including `Block NUmber`)
  - Type mismatch: Airtable `Value` configured as number, but workflow supplies a string; with “attemptToConvertTypes: false”, Airtable may reject or coerce depending on API behavior.
  - Duplicate tx hashes: workflow does not deduplicate; re-running for the same block can create duplicate rows.

#### Node: Genrate transction summary
- **Type / role:** Code (`n8n-nodes-base.code`) — consolidates Airtable create outputs into one Slack-ready message.
- **Configuration (interpreted):**
  - Filters to items where `item.json.fields.Value` exists (expects Airtable response format)
  - If none: returns `{ text: "No high-value transfers found." }`
  - Otherwise:
    - counts transfers saved (`validItems.length`)
    - finds largest by `BigInt(fields.Value)`
    - produces message:
      - `"{N} high-value transfers detected. Largest transfer: {Value} from {From} to {To}. Stored in Airtable."`
- **Connections:**
  - **Input ←** `Save transaction`
  - **Output →** `Send transaction alert`
- **Version-specific notes:** TypeVersion `2`.
- **Edge cases / failures:**
  - If Airtable output doesn’t include `fields` (e.g., error payload), filter may produce zero and send “No high-value transfers found.”
  - `fields.Value` may be a number, not string; `BigInt(123)` is valid, but mixed formats can still cause issues if decimal values appear.
  - This node only summarizes **records that were successfully created**, not raw blockchain findings.

#### Node: Send transaction alert
- **Type / role:** Slack (`n8n-nodes-base.slack`) — posts message to a channel.
- **Configuration (interpreted):**
  - Sends text:
    - `High value transfer alert`
    - followed by `{{$json.text}}` from the summary node
  - Channel: `n8n` (ID: `C09S57E2JQ2`)
  - Does not include link back to workflow execution.
- **Credentials:**
  - Slack API credential (`slackApi`).
- **Connections:**
  - **Input ←** `Genrate transction summary`
  - **Output →** none (end)
- **Version-specific notes:** TypeVersion `2.3`.
- **Edge cases / failures:**
  - Missing Slack scopes (chat:write), channel access issues
  - Posting to a private channel requires the app/bot to be invited
  - Slack rate limiting (unlikely for one message/day)

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Daily Check | Schedule Trigger | Daily entry point | — | Get latest blocknumber | Runs the workflow automatically at a set time every day. |
| Get latest blocknumber | HTTP Request | JSON-RPC: fetch latest Ethereum block number | Daily Check | Fetch USDT transaction logs | ## High-Value USDT Transaction Detection<br><br>**Get latest blocknumber** – Gets the latest Ethereum block number from the blockchain.<br><br>**Fetch USDT transaction logs** – Fetches all USDT “Transfer” events from that block.<br><br>**Extract transactions details** – Extracts and formats sender, receiver, amount, and transaction details from the blockchain data.<br><br>**Filter high value transaction** – Checks if the transaction amount is above the threshold to consider it “high-value.” |
| Fetch USDT transaction logs | HTTP Request | JSON-RPC: fetch USDT Transfer logs for latest block | Get latest blocknumber | Extract transactions details | ## High-Value USDT Transaction Detection<br><br>**Get latest blocknumber** – Gets the latest Ethereum block number from the blockchain.<br><br>**Fetch USDT transaction logs** – Fetches all USDT “Transfer” events from that block.<br><br>**Extract transactions details** – Extracts and formats sender, receiver, amount, and transaction details from the blockchain data.<br><br>**Filter high value transaction** – Checks if the transaction amount is above the threshold to consider it “high-value.” |
| Extract transactions details | Code | Parse logs → structured tx items | Fetch USDT transaction logs | Filter high value transaction | ## High-Value USDT Transaction Detection<br><br>**Get latest blocknumber** – Gets the latest Ethereum block number from the blockchain.<br><br>**Fetch USDT transaction logs** – Fetches all USDT “Transfer” events from that block.<br><br>**Extract transactions details** – Extracts and formats sender, receiver, amount, and transaction details from the blockchain data.<br><br>**Filter high value transaction** – Checks if the transaction amount is above the threshold to consider it “high-value.” |
| Filter high value transaction | IF | Keep only transfers above threshold | Extract transactions details | Save transaction | ## High-Value USDT Transaction Detection<br><br>**Get latest blocknumber** – Gets the latest Ethereum block number from the blockchain.<br><br>**Fetch USDT transaction logs** – Fetches all USDT “Transfer” events from that block.<br><br>**Extract transactions details** – Extracts and formats sender, receiver, amount, and transaction details from the blockchain data.<br><br>**Filter high value transaction** – Checks if the transaction amount is above the threshold to consider it “high-value.” |
| Save transaction | Airtable | Store high-value transfers | Filter high value transaction | Genrate transction summary | ## Record & Alert High-Value Transactions<br><br>**Save transaction** – Saves the high-value transaction details into Airtable for tracking.<br><br>**Genrate transaction summary** – Summarizes all saved transactions and identifies the largest one.<br><br>**Send transaction alert** – Sends a clear alert to Slack with the number of high-value transactions and the largest transfer. |
| Genrate transction summary | Code | Summarize saved transfers into one message | Save transaction | Send transaction alert | ## Record & Alert High-Value Transactions<br><br>**Save transaction** – Saves the high-value transaction details into Airtable for tracking.<br><br>**Genrate transaction summary** – Summarizes all saved transactions and identifies the largest one.<br><br>**Send transaction alert** – Sends a clear alert to Slack with the number of high-value transactions and the largest transfer. |
| Send transaction alert | Slack | Send Slack notification | Genrate transction summary | — | ## Record & Alert High-Value Transactions<br><br>**Save transaction** – Saves the high-value transaction details into Airtable for tracking.<br><br>**Genrate transaction summary** – Summarizes all saved transactions and identifies the largest one.<br><br>**Send transaction alert** – Sends a clear alert to Slack with the number of high-value transactions and the largest transfer. |
| Sticky Note | Sticky Note | Documentation / instructions | — | — | ## How it works<br>This workflow automatically checks the blockchain every day and looks for big USDT transactions happening in the latest block. It extracts the sender, receiver, amount, and transaction details. If the transaction value is higher than a defined limit, the workflow saves those records into Airtable for tracking. After saving, it creates a short summary and sends one clear alert message to Slack. That message tells the team how many large transactions were found and highlights the biggest one. This helps you stay updated without checking the blockchain manually or receiving too many notifications.<br><br>## Setup steps<br><br>**1.** Add your Alchemy API URL for Ethereum mainnet in the HTTP nodes.<br><br>**2.** Set the USDT contract address and Transfer event topic (already included).<br><br>**3.** Connect your Airtable account and select the correct base and table.<br><br>**4.** Make sure fields in Airtable match the workflow: Contract, From Address, To Address, Value, Block Number, and txHash.<br><br>**5.** Connect your Slack account and select the channel where you want alerts.<br><br>**6.** Adjust the value threshold in the IF node if you want a different limit.<br><br>**7.** Activate the workflow to start receiving daily notifications. |
| Sticky Note1 | Sticky Note | Comment: schedule purpose | — | — | Runs the workflow automatically at a set time every day. |
| Sticky Note2 | Sticky Note | Comment: detection block description | — | — | ## High-Value USDT Transaction Detection<br><br>**Get latest blocknumber** – Gets the latest Ethereum block number from the blockchain.<br><br>**Fetch USDT transaction logs** – Fetches all USDT “Transfer” events from that block.<br><br>**Extract transactions details** – Extracts and formats sender, receiver, amount, and transaction details from the blockchain data.<br><br>**Filter high value transaction** – Checks if the transaction amount is above the threshold to consider it “high-value.” |
| Sticky Note3 | Sticky Note | Comment: record+alert block description | — | — | ## Record & Alert High-Value Transactions<br><br>**Save transaction** – Saves the high-value transaction details into Airtable for tracking.<br><br>**Genrate transaction summary** – Summarizes all saved transactions and identifies the largest one.<br><br>**Send transaction alert** – Sends a clear alert to Slack with the number of high-value transactions and the largest transfer. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add node: “Schedule Trigger”**
   - Name: `Daily Check`
   - Configure to run **every day at 10:00** (set the hour in the schedule rule).
3. **Add node: “HTTP Request”**
   - Name: `Get latest blocknumber`
   - Method: `POST`
   - URL: `https://eth-mainnet.g.alchemy.com/v2/<YOUR_ALCHEMY_API_KEY>`
   - Body type: JSON
   - Body:
     - `jsonrpc: "2.0"`, `id: 1`, `method: "eth_blockNumber"`, `params: []`
   - Header: `Content-Type: application/json`
4. **Connect:** `Daily Check` → `Get latest blocknumber`.
5. **Add node: “HTTP Request”**
   - Name: `Fetch USDT transaction logs`
   - Method: `POST`
   - URL: `https://eth-mainnet.g.alchemy.com/v2/<YOUR_ALCHEMY_API_KEY>`
   - Body type: Raw JSON (or JSON, but ensure expressions resolve correctly)
   - Set:
     - `fromBlock` = expression `{{ $json.result }}`
     - `toBlock` = expression `{{ $json.result }}`
     - `address` = `0xdAC17F958D2ee523a2206206994597C13D831ec7`
     - `topics[0]` = `0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef`
   - Header: `Content-Type: application/json`
6. **Connect:** `Get latest blocknumber` → `Fetch USDT transaction logs`.
7. **Add node: “Code”**
   - Name: `Extract transactions details`
   - Paste JS logic that:
     - iterates over `$json.result` array
     - extracts `from/to` from `topics[1]/topics[2]`
     - converts `data` to integer string using `BigInt`
     - outputs items with: `contract, from, to, value, blockNumber, txHash`
8. **Connect:** `Fetch USDT transaction logs` → `Extract transactions details`.
9. **Add node: “IF”**
   - Name: `Filter high value transaction`
   - Condition: Number `{{$json.value}}` **greater than** `1000000000`
   - (This equals ~**1,000 USDT** in raw 6-decimal units.)
10. **Connect:** `Extract transactions details` → `Filter high value transaction` (main).
11. **Add node: “Airtable”**
   - Name: `Save transaction`
   - **Credentials:** create/connect an **Airtable Personal Access Token** credential.
   - Operation: **Create**
   - Select Base: `n8n Demo` (or your base)
   - Select Table: `Blockchain` (or your table)
   - Map fields (must exist in Airtable with matching names):
     - `Contract` = `{{$json.contract}}`
     - `From Address` = `{{$json.from}}`
     - `To Address` = `{{$json.to}}`
     - `Value` = `{{$json.value}}`
     - `Block NUmber` = `{{$json.blockNumber}}` (match your exact Airtable field name)
     - `txHash` = `{{$json.txHash}}`
12. **Connect:** `Filter high value transaction` (true output) → `Save transaction`.
13. **Add node: “Code”**
   - Name: `Genrate transction summary`
   - Implement logic that:
     - reads Airtable output items (`item.json.fields`)
     - counts them
     - selects largest by `BigInt(fields.Value)`
     - outputs a single item `{ text: "..." }`
14. **Connect:** `Save transaction` → `Genrate transction summary`.
15. **Add node: “Slack”**
   - Name: `Send transaction alert`
   - **Credentials:** connect Slack API (bot/token) with permission to post messages.
   - Resource/operation: send a message to a channel
   - Channel: select your target channel (e.g., `#n8n`)
   - Text:
     - `High value transfer alert`
     - `{{ $json.text }}`
   - Disable “include link to workflow” (optional, matches JSON).
16. **Connect:** `Genrate transction summary` → `Send transaction alert`.
17. **Verify Airtable schema**
   - Ensure fields exist and types match (notably `Value` number, and consistent naming like `Block NUmber` vs `Block Number`).
18. **Activate the workflow**.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| USDT contract address used: `0xdAC17F958D2ee523a2206206994597C13D831ec7` | Hardcoded in “Fetch USDT transaction logs” |
| ERC-20 Transfer topic used: `0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef` | Hardcoded in “Fetch USDT transaction logs” |
| Threshold `1000000000` is in **raw USDT units (6 decimals)** → about **1,000 USDT** | IF node comparison logic |
| Workflow only checks the **latest block at run time**, not a range of blocks | Both fromBlock and toBlock set to latest block number |
| If no transfers exceed threshold, workflow will not store anything and (as wired) will not send Slack (because IF false path is not connected). | Connection design consideration |
| Sticky note “How it works” includes setup checklist (Alchemy URL, Airtable fields, Slack channel, threshold, activation). | Embedded workflow documentation |
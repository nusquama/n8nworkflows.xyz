Manage Supabase database records with Telegram bot commands

https://n8nworkflows.xyz/workflows/manage-supabase-database-records-with-telegram-bot-commands-13166


# Manage Supabase database records with Telegram bot commands

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Manage Supabase database records with Telegram bot commands  
**Workflow name (in JSON):** Manage Supabase database with Telegram commands  
**Purpose:** Turns a Telegram bot into a lightweight CRUD interface for a Supabase table (`products`). Authorized Telegram users can add, list, get, update, delete, and search products by sending bot commands.

### 1.1 Input Reception (Telegram)
Receives Telegram messages via webhook polling/updates and normalizes key fields (chatId, text, firstName) for downstream logic.

### 1.2 Authorization Gate
Checks the sender chat ID against an allowlist. Unauthorized messages are silently dropped (no response path configured).

### 1.3 Command Parsing & Routing
Parses `/command` and parameters into a structured JSON payload, then routes execution to the correct handler using a Switch.

### 1.4 Command Handlers (Supabase CRUD + responses)
For each command, validates parsing (error flag), performs the Supabase operation, formats results (where needed), and replies via Telegram.

Logical handlers:
- **Add**: `/add name, price, quantity, category`
- **List**: `/list [category]`
- **Get**: `/get id`
- **Update**: `/update id field=value field=value`
- **Delete**: `/delete id`
- **Search**: `/search text`
- **Help**: `/help`
- **Fallback**: unknown command

---

## 2. Block-by-Block Analysis

### Block A — Input Reception & Normalization
**Overview:** Captures incoming Telegram messages and extracts only the fields needed by the rest of the workflow.  
**Nodes involved:** `Telegram Trigger`, `Extract Message Data`

#### Node: Telegram Trigger
- **Type / role:** `telegramTrigger` — entry point; listens to Telegram updates of type `message`.
- **Configuration choices:**
  - Updates: `message` only (ignores callbacks, edited messages, etc.).
- **Key data used:** Expects `message.chat.id`, `message.text`, `message.from.first_name`.
- **Outputs:** One item per message update.
- **Connections:** → `Extract Message Data`
- **Potential failures / edge cases:**
  - Telegram credential/token invalid, webhook misconfigured, or network errors.
  - Non-text messages: `message.text` may be undefined (photos, stickers). Downstream parsing defaults to empty string but may route to “unknown command”.

#### Node: Extract Message Data
- **Type / role:** `Set` — normalizes message data into flat fields.
- **Configuration choices:**
  - Creates:
    - `chatId` (number) = `{{$json.message.chat.id}}`
    - `text` (string) = `{{$json.message.text}}`
    - `firstName` (string) = `{{$json.message.from.first_name}}`
- **Connections:** ← `Telegram Trigger`; → `Is Authorized?`
- **Potential failures / edge cases:**
  - If `message.text` missing, `text` becomes `null/undefined` (then parser treats as `''`).
  - If Telegram payload differs (channels, anonymous admins), `from.first_name` may be missing.

**Sticky note applying to this area (contextual):**
- “### 🔐 Authorization …” (also relevant to the next block)

---

### Block B — Authorization Gate
**Overview:** Allows only specific Telegram chat IDs to proceed; all others stop with no reply.  
**Nodes involved:** `Is Authorized?`

#### Node: Is Authorized?
- **Type / role:** `If` — allowlist gate.
- **Configuration choices:**
  - OR conditions:
    - `{{$json.chatId}} == 123456789`
    - `{{$json.chatId}} == 987654321`
  - Case sensitivity irrelevant (numeric).
- **Connections:**
  - True output → `Parse Command and Parameters`
  - False output → (not connected) ⇒ workflow ends silently
- **Potential failures / edge cases:**
  - Mis-typed chat IDs will lock out valid users.
  - Groups vs private chats: group chat IDs can be negative; ensure you authorize the correct chat ID type.
  - No “unauthorized” response is sent; this is intentional but can confuse users.

**Sticky note (covers this block):**
- “### 🔐 Authorization  
  Extracts chat ID from message and validates against authorized list using OR conditions. To add more users, edit the If node and add more conditions.”

---

### Block C — Command Parsing & Routing
**Overview:** Converts message text into `{command, chatId, rawParams, params}` and routes to the correct handler output.  
**Nodes involved:** `Parse Command and Parameters`, `Route by Command`

#### Node: Parse Command and Parameters
- **Type / role:** `Code` — parses Telegram command syntax into structured parameters.
- **Configuration choices (behavior):**
  - Extracts command via regex: `^\/([a-z]+)` (letters only).
  - `command` becomes lowercase; if no match → `unknown`.
  - `rawParams` is remaining text after the command.
  - Command-specific parsing:
    - **add**: comma-separated `name, price, quantity, category` (>= 4 parts required)
    - **list**: optional category string
    - **get/delete**: integer id
    - **update**: space-separated: first token id, remaining tokens `field=value`
      - `price` parsed as float; `quantity` as int; others as string
      - requires at least one valid update field
    - **search**: free text required
    - **help**: no params
    - default: sets `params.error` = unknown command message
- **Key variables referenced:** `$input.first().json.text`, `$input.first().json.chatId`
- **Outputs:** Single item: `{ command, chatId, rawParams, params }`
- **Connections:** → `Route by Command`
- **Potential failures / edge cases:**
  - `/add` with commas inside product name will break parsing.
  - `/update` values cannot contain spaces (split by space); e.g., `name=New iPhone` will be truncated.
  - `/command@BotName` (common in groups) is not supported by regex and will become `unknown` unless adapted.
  - Non-numeric IDs parse to `null` and raise format errors.

#### Node: Route by Command
- **Type / role:** `Switch` — routes execution based on `command`.
- **Configuration choices:**
  - Outputs (renamed): `add`, `list`, `get`, `update`, `delete`, `search`, `help`
  - Fallback output set to `extra` (used for unknown or non-matching).
  - Each rule checks: `{{$json.command}} equals "<command>"`
- **Connections:**
  - `add` → `Add Has Error?`
  - `list` → `Filter by Category?`
  - `get` → `Get Has Error?`
  - `update` → `Update Has Error?`
  - `delete` → `Delete Has Error?`
  - `search` → `Search Has Error?`
  - `help` → `Send Help Message`
  - `extra` → `Send Unknown Command`
- **Potential failures / edge cases:**
  - If parsing outputs unexpected command strings, it will hit fallback.

**Sticky note (covers this block):**
- “### 🔀 Command Router  
  Parses the message to extract the command and routes to the appropriate handler. Supports: /add, /list, /get, /update, /delete, /search, /help”

---

### Block D — Add Product Handler (`/add`)
**Overview:** Validates `/add` format; inserts a row into Supabase; replies with inserted record fields or an error message.  
**Nodes involved:** `Add Has Error?`, `Send Add Error`, `Supabase Insert Product`, `Send Add Success`

#### Node: Add Has Error?
- **Type / role:** `If` — checks if parsing produced `params.error`.
- **Condition:** “string exists” on `{{$json.params.error}}`
- **Connections:**
  - True → `Send Add Error`
  - False → `Supabase Insert Product`
- **Potential failures / edge cases:**
  - If `params.error` is set but empty string, `exists` behavior depends on n8n’s interpretation; here it will typically treat non-null as existing.

#### Node: Send Add Error
- **Type / role:** `Telegram` — sends format guidance.
- **Key fields:**
  - `chatId` = `{{$json.chatId}}`
  - Text includes `{{$json.params.error}}`
- **Connections:** terminal
- **Potential failures:**
  - Telegram API errors; invalid chat ID.

#### Node: Supabase Insert Product
- **Type / role:** `Supabase` — inserts into `products`.
- **Configuration choices:**
  - Table: `products`
  - Operation: default insert (implied by node config; fields provided)
  - Field values mapped from parsed params:
    - name, price, quantity, category
- **Connections:** → `Send Add Success`
- **Potential failures / edge cases:**
  - Supabase credentials/URL/key invalid.
  - Table/columns mismatch (schema not aligned with node fields).
  - `price`/`quantity` types incompatible with DB column types.
  - RLS policies blocking insert (common in Supabase).

#### Node: Send Add Success
- **Type / role:** `Telegram` — confirms insertion and echoes record details.
- **Key expressions:**
  - `chatId` = `{{ $('Parse Command and Parameters').item.json.chatId }}`
  - Uses inserted row fields: `{{$json.name}}`, `{{$json.price}}`, etc.
- **Connections:** terminal
- **Potential failures / edge cases:**
  - If Supabase node returns a different structure (e.g., wraps in `data`), these `{{$json.field}}` references may break.

---

### Block E — List Products Handler (`/list [category]`)
**Overview:** Lists all products, optionally filtering by category; formats into a single Telegram message.  
**Nodes involved:** `Filter by Category?`, `Supabase List Filtered`, `Supabase List All`, `Format Product List`, `Send Product List`

#### Node: Filter by Category?
- **Type / role:** `If` — checks if `params.category` exists.
- **Condition:** “string exists” on `{{$json.params.category}}`
- **Connections:**
  - True → `Supabase List Filtered`
  - False → `Supabase List All`
- **Edge cases:**
  - If category is an empty string, behavior depends on “exists” semantics.

#### Node: Supabase List Filtered
- **Type / role:** `Supabase` — `getAll` with filter.
- **Configuration:**
  - Operation: `getAll`, `returnAll: true`
  - Filter string: `category=eq.{{ $json.params.category }}`
- **Connections:** → `Format Product List`
- **Potential failures:**
  - Filter syntax issues if category contains special characters; may require URL encoding depending on implementation.
  - RLS may block select.

#### Node: Supabase List All
- **Type / role:** `Supabase` — `getAll` without filters.
- **Configuration:** returnAll = true
- **Connections:** → `Format Product List`

#### Node: Format Product List
- **Type / role:** `Code` — builds a single message for Telegram from multiple input items.
- **Key variables:**
  - `items = $input.all()` (each is a product record)
  - `chatId` + `category` read from `Parse Command and Parameters`
- **Behavior:**
  - If no items: returns “No products…” message.
  - Else enumerates products with ID, name, price, quantity, category.
- **Connections:** → `Send Product List`
- **Edge cases:**
  - Large product lists can exceed Telegram message length limits (~4096 chars). No pagination implemented.

#### Node: Send Product List
- **Type / role:** `Telegram` — sends the formatted list.
- **Key fields:**
  - `chatId` = `{{$json.chatId}}`
  - `text` = `{{$json.message}}`
- **Connections:** terminal

---

### Block F — Get Product Handler (`/get id`)
**Overview:** Validates id; retrieves a product by id; replies with details.  
**Nodes involved:** `Get Has Error?`, `Send Get Error`, `Supabase Get Product`, `Send Product Details`

#### Node: Get Has Error?
- **Type / role:** `If` — checks `params.error`.
- **Connections:** True → `Send Get Error`; False → `Supabase Get Product`

#### Node: Send Get Error
- **Type / role:** `Telegram` — sends correct usage.
- **chatId:** `{{$json.chatId}}`

#### Node: Supabase Get Product
- **Type / role:** `Supabase` — `getAll` filtered by `id`.
- **Configuration:**
  - Operation: `getAll`
  - Filter: `id=eq.{{ $json.params.id }}`
  - `limit: 1`
- **Connections:** → `Send Product Details`
- **Edge cases:**
  - If no record found, Supabase may return **zero items**; then `Send Product Details` will not run (no item to send). There is no “not found” message path.

#### Node: Send Product Details
- **Type / role:** `Telegram` — sends record fields.
- **chatId:** `{{ $('Parse Command and Parameters').item.json.chatId }}`
- **Edge cases:**
  - If multiple items returned, message is sent per item (though limited to 1 here).

---

### Block G — Update Product Handler (`/update id field=value ...`)
**Overview:** Validates update format; maps update fields; updates Supabase row by id; confirms success.  
**Nodes involved:** `Update Has Error?`, `Send Update Error`, `Prepare Update Data`, `Supabase Update Product`, `Send Update Success`

#### Node: Update Has Error?
- **Type / role:** `If` — checks `params.error`.
- **Connections:** True → `Send Update Error`; False → `Prepare Update Data`

#### Node: Send Update Error
- **Type / role:** `Telegram` — usage instructions and valid fields.
- **chatId:** `{{$json.chatId}}`

#### Node: Prepare Update Data
- **Type / role:** `Set` — prepares a flat structure for the update node.
- **Configuration choices:**
  - `ignoreConversionErrors: true` to avoid failing when values are undefined.
  - Sets:
    - `id = {{$json.params.id}}`
    - `name/price/quantity/category` from `{{$json.params.updates.<field>}}`
- **Connections:** → `Supabase Update Product`
- **Edge cases:**
  - Fields not provided become `undefined`. Depending on Supabase node behavior, undefined fields might:
    - be ignored, or
    - overwrite with null (implementation-dependent). Validate in your n8n version.

#### Node: Supabase Update Product
- **Type / role:** `Supabase` — updates `products` where `id` matches.
- **Configuration:**
  - Operation: `update`
  - Filter: `id=eq.{{ $json.id }}`
- **Connections:** → `Send Update Success`
- **Potential failures:**
  - No row updated (id not found) may still return success-like output depending on API response; no explicit check exists.
  - RLS can block update.

#### Node: Send Update Success
- **Type / role:** `Telegram` — confirms update.
- **chatId:** `{{ $('Parse Command and Parameters').item.json.chatId }}`

---

### Block H — Delete Product Handler (`/delete id`)
**Overview:** Validates id; deletes product; confirms deletion.  
**Nodes involved:** `Delete Has Error?`, `Send Delete Error`, `Supabase Delete Product`, `Send Delete Success`

#### Node: Delete Has Error?
- **Type / role:** `If` — checks `params.error`.
- **Connections:** True → `Send Delete Error`; False → `Supabase Delete Product`

#### Node: Send Delete Error
- **Type / role:** `Telegram` — usage instructions.
- **chatId:** `{{$json.chatId}}`

#### Node: Supabase Delete Product
- **Type / role:** `Supabase` — deletes from `products` by id.
- **Configuration:**
  - Operation: `delete`
  - Filter: `id=eq.{{ $json.params.id }}`
- **Connections:** → `Send Delete Success`
- **Edge cases:**
  - If id doesn’t exist, delete may affect 0 rows; workflow still sends “deleted successfully” (no verification).

#### Node: Send Delete Success
- **Type / role:** `Telegram` — confirms deletion.
- **Key expression:** embeds ID from parser: `{{ $('Parse Command and Parameters').item.json.params.id }}`
- **chatId:** `{{ $('Parse Command and Parameters').item.json.chatId }}`

---

### Block I — Search Products Handler (`/search text`)
**Overview:** Validates query; searches products by name (ILIKE); formats results; replies.  
**Nodes involved:** `Search Has Error?`, `Send Search Error`, `Supabase Search Products`, `Format Search Results`, `Send Search Results`

#### Node: Search Has Error?
- **Type / role:** `If` — checks `params.error`.
- **Connections:** True → `Send Search Error`; False → `Supabase Search Products`

#### Node: Send Search Error
- **Type / role:** `Telegram` — usage instructions.
- **chatId:** `{{$json.chatId}}`

#### Node: Supabase Search Products
- **Type / role:** `Supabase` — `getAll` with `ilike` filter.
- **Configuration:**
  - Filter: `name=ilike.*{{ $json.params.searchText }}*`
  - returnAll = true
- **Connections:** → `Format Search Results`
- **Edge cases:**
  - Special characters in `searchText` may affect filter parsing.
  - Large result sets may exceed message length limits.

#### Node: Format Search Results
- **Type / role:** `Code` — builds response string (similar to list formatter).
- **Connections:** → `Send Search Results`

#### Node: Send Search Results
- **Type / role:** `Telegram` — sends formatted search results.
- **chatId:** `{{$json.chatId}}`

---

### Block J — Help & Fallback
**Overview:** Sends a Markdown help message or an “unknown command” reply.  
**Nodes involved:** `Send Help Message`, `Send Unknown Command`

#### Node: Send Help Message
- **Type / role:** `Telegram` — sends command reference.
- **Configuration choices:**
  - `parse_mode: Markdown`
  - `chatId: {{$json.chatId}}`
- **Edge cases:**
  - Markdown formatting can break if you later inject unescaped user content (currently static text).

#### Node: Send Unknown Command
- **Type / role:** `Telegram` — simple fallback message.
- **chatId:** `{{$json.chatId}}`

---

### Block K — Documentation Sticky Notes (non-executing)
**Overview:** Embedded notes describing setup, SQL schema, and customization guidance.  
**Nodes involved:** `Sticky Note`, `Sticky Note1`, `Sticky Note2`  
- These nodes do not affect execution.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Telegram Trigger | telegramTrigger | Entry point: receive Telegram messages | — | Extract Message Data | ## How it works… (full note includes setup + SQL + customization) |
| Extract Message Data | set | Normalize Telegram payload into `chatId/text/firstName` | Telegram Trigger | Is Authorized? | ### 🔐 Authorization… |
| Is Authorized? | if | Allowlist gate by chat ID | Extract Message Data | Parse Command and Parameters (true) | ### 🔐 Authorization… |
| Parse Command and Parameters | code | Parse `/command` and params into structured object | Is Authorized? | Route by Command | ### 🔀 Command Router… |
| Route by Command | switch | Route to handler per command | Parse Command and Parameters | Add Has Error?, Filter by Category?, Get Has Error?, Update Has Error?, Delete Has Error?, Search Has Error?, Send Help Message, Send Unknown Command | ### 🔀 Command Router… |
| Add Has Error? | if | Validate `/add` parsing | Route by Command (add) | Send Add Error (true), Supabase Insert Product (false) |  |
| Send Add Error | telegram | Reply with `/add` format error | Add Has Error? (true) | — |  |
| Supabase Insert Product | supabase | Insert product row | Add Has Error? (false) | Send Add Success |  |
| Send Add Success | telegram | Confirm insert with returned fields | Supabase Insert Product | — |  |
| Filter by Category? | if | Decide filtered vs full list | Route by Command (list) | Supabase List Filtered (true), Supabase List All (false) |  |
| Supabase List Filtered | supabase | Fetch products by category | Filter by Category? (true) | Format Product List |  |
| Supabase List All | supabase | Fetch all products | Filter by Category? (false) | Format Product List |  |
| Format Product List | code | Build list message from items | Supabase List Filtered / Supabase List All | Send Product List |  |
| Send Product List | telegram | Send list message | Format Product List | — |  |
| Get Has Error? | if | Validate `/get` parsing | Route by Command (get) | Send Get Error (true), Supabase Get Product (false) |  |
| Send Get Error | telegram | Reply with `/get` format error | Get Has Error? (true) | — |  |
| Supabase Get Product | supabase | Fetch product by id | Get Has Error? (false) | Send Product Details |  |
| Send Product Details | telegram | Send product detail message | Supabase Get Product | — |  |
| Update Has Error? | if | Validate `/update` parsing | Route by Command (update) | Send Update Error (true), Prepare Update Data (false) |  |
| Send Update Error | telegram | Reply with `/update` format error | Update Has Error? (true) | — |  |
| Prepare Update Data | set | Map update fields into flat JSON | Update Has Error? (false) | Supabase Update Product |  |
| Supabase Update Product | supabase | Update product by id | Prepare Update Data | Send Update Success |  |
| Send Update Success | telegram | Confirm update | Supabase Update Product | — |  |
| Delete Has Error? | if | Validate `/delete` parsing | Route by Command (delete) | Send Delete Error (true), Supabase Delete Product (false) |  |
| Send Delete Error | telegram | Reply with `/delete` format error | Delete Has Error? (true) | — |  |
| Supabase Delete Product | supabase | Delete product by id | Delete Has Error? (false) | Send Delete Success |  |
| Send Delete Success | telegram | Confirm deletion | Supabase Delete Product | — |  |
| Search Has Error? | if | Validate `/search` parsing | Route by Command (search) | Send Search Error (true), Supabase Search Products (false) |  |
| Send Search Error | telegram | Reply with `/search` format error | Search Has Error? (true) | — |  |
| Supabase Search Products | supabase | Search by name using ILIKE | Search Has Error? (false) | Format Search Results |  |
| Format Search Results | code | Build search results message | Supabase Search Products | Send Search Results |  |
| Send Search Results | telegram | Send search message | Format Search Results | — |  |
| Send Help Message | telegram | Send Markdown help | Route by Command (help) | — |  |
| Send Unknown Command | telegram | Fallback for unknown commands | Route by Command (extra) | — |  |
| Sticky Note | stickyNote | Documentation (setup/SQL/customization) | — | — | ## How it works… (full note includes setup + SQL + customization) |
| Sticky Note1 | stickyNote | Documentation (authorization) | — | — | ### 🔐 Authorization… |
| Sticky Note2 | stickyNote | Documentation (router) | — | — | ### 🔀 Command Router… |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a Telegram bot**
   1) In Telegram, open **@BotFather** → `/newbot`  
   2) Save the **bot token**.

2. **Create Supabase table**
   1) In Supabase SQL editor, create the table:
      - Table name: `products`
      - Columns: `id` (serial PK), `name` (text not null), `price` (decimal), `quantity` (int default 0), `category` (text), `created_at` (timestamp default now)
   2) Ensure Row Level Security (RLS) policies allow the operations you expect (select/insert/update/delete) for your API key type.

3. **Set up credentials in n8n**
   1) **Telegram credential**: add bot token (Telegram API).
   2) **Supabase credential**: set Supabase URL and API key (service role key if you want to bypass RLS; otherwise configure policies appropriately).

4. **Build nodes and connect them in order**

5. **Node 1: Telegram Trigger**
   - Add node: **Telegram Trigger**
   - Updates: `message`
   - Connect to **Extract Message Data**

6. **Node 2: Extract Message Data (Set)**
   - Add node: **Set**
   - Add fields:
     - `chatId` (Number): `{{$json.message.chat.id}}`
     - `text` (String): `{{$json.message.text}}`
     - `firstName` (String): `{{$json.message.from.first_name}}`
   - Connect to **Is Authorized?**

7. **Node 3: Is Authorized? (If)**
   - Add node: **If**
   - Condition group: **OR**
   - Add conditions (Number → equals):
     - `{{$json.chatId}}` equals `123456789`
     - `{{$json.chatId}}` equals `987654321`
   - True output → **Parse Command and Parameters**
   - Leave False output unconnected (or optionally add a “not authorized” reply node if desired).

8. **Node 4: Parse Command and Parameters (Code)**
   - Add node: **Code**
   - Paste the parsing logic (supporting add/list/get/update/delete/search/help).
   - Output should be a single item with:
     - `command`, `chatId`, `rawParams`, `params` (+ optional `params.error`)
   - Connect to **Route by Command**

9. **Node 5: Route by Command (Switch)**
   - Add node: **Switch**
   - Add rules on `{{$json.command}}` equals:
     - `add`, `list`, `get`, `update`, `delete`, `search`, `help`
   - Enable fallback output (named “extra”)
   - Connect outputs:
     - add → Add Has Error?
     - list → Filter by Category?
     - get → Get Has Error?
     - update → Update Has Error?
     - delete → Delete Has Error?
     - search → Search Has Error?
     - help → Send Help Message
     - extra → Send Unknown Command

10. **Add handler (`/add`)**
   1) **Add Has Error? (If)**: checks existence of `{{$json.params.error}}`
      - True → Send Add Error
      - False → Supabase Insert Product
   2) **Send Add Error (Telegram)**:
      - chatId: `{{$json.chatId}}`
      - text: include `{{$json.params.error}}` and correct format
   3) **Supabase Insert Product (Supabase)**:
      - Table: `products`
      - Insert fields: name/price/quantity/category mapped from `{{$json.params.<field>}}`
   4) **Send Add Success (Telegram)**:
      - chatId: `{{ $('Parse Command and Parameters').item.json.chatId }}`
      - text uses inserted row fields: `{{$json.id}}`, `{{$json.name}}`, etc.

11. **List handler (`/list`)**
   1) **Filter by Category? (If)**: check `{{$json.params.category}}` exists
      - True → Supabase List Filtered
      - False → Supabase List All
   2) **Supabase List Filtered (Supabase)**:
      - Operation: getAll, returnAll true
      - Filter string: `category=eq.{{ $json.params.category }}`
   3) **Supabase List All (Supabase)**:
      - Operation: getAll, returnAll true
   4) **Format Product List (Code)**:
      - Build single `message` string; set `chatId`
   5) **Send Product List (Telegram)**:
      - chatId: `{{$json.chatId}}`
      - text: `{{$json.message}}`

12. **Get handler (`/get`)**
   1) **Get Has Error? (If)**: if `params.error` exists
      - True → Send Get Error
      - False → Supabase Get Product
   2) **Supabase Get Product (Supabase)**:
      - Operation: getAll
      - Filter: `id=eq.{{ $json.params.id }}`
      - Limit: 1
   3) **Send Product Details (Telegram)**:
      - chatId: `{{ $('Parse Command and Parameters').item.json.chatId }}`
      - text uses `{{$json.id}}`, `{{$json.name}}`, etc.

13. **Update handler (`/update`)**
   1) **Update Has Error? (If)** → Send Update Error or Prepare Update Data
   2) **Prepare Update Data (Set)**:
      - ignoreConversionErrors: enabled
      - set `id` and updateable fields from `params.updates`
   3) **Supabase Update Product (Supabase)**:
      - Operation: update
      - Filter: `id=eq.{{ $json.id }}`
   4) **Send Update Success (Telegram)**:
      - chatId: `{{ $('Parse Command and Parameters').item.json.chatId }}`

14. **Delete handler (`/delete`)**
   1) **Delete Has Error? (If)** → Send Delete Error or Supabase Delete Product
   2) **Supabase Delete Product (Supabase)**:
      - Operation: delete
      - Filter: `id=eq.{{ $json.params.id }}`
   3) **Send Delete Success (Telegram)**:
      - chatId: `{{ $('Parse Command and Parameters').item.json.chatId }}`

15. **Search handler (`/search`)**
   1) **Search Has Error? (If)** → Send Search Error or Supabase Search Products
   2) **Supabase Search Products (Supabase)**:
      - Operation: getAll, returnAll true
      - Filter: `name=ilike.*{{ $json.params.searchText }}*`
   3) **Format Search Results (Code)** → build `message` and `chatId`
   4) **Send Search Results (Telegram)** → send `{{$json.message}}`

16. **Help & fallback**
   - **Send Help Message (Telegram)**:
     - chatId: `{{$json.chatId}}`
     - parse_mode: Markdown
     - static text listing commands
   - **Send Unknown Command (Telegram)**:
     - chatId: `{{$json.chatId}}`
     - text: “Use /help…”

17. **Activate workflow**
   - Save and activate.
   - In Telegram, send `/help` from an authorized chat.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “This workflow turns Telegram into a database management interface for Supabase… Receive message → Validate user → Parse command → Route & execute → Respond” | Sticky note “How it works” |
| Setup steps: create bot via @BotFather, configure credentials, create Supabase project, create `products` table, add chat IDs, test `/help` | Sticky note “Setup steps” |
| Supabase schema SQL for `products` table (id, name, price, quantity, category, created_at) | Sticky note “Supabase table SQL” |
| Customization: change table name in all Supabase nodes; update parsing + mappings + formatting; add authorized users by adding OR conditions | Sticky note “Customization Guide” |
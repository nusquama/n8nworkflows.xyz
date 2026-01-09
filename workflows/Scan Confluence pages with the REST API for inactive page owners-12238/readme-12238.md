Scan Confluence pages with the REST API for inactive page owners

https://n8nworkflows.xyz/workflows/scan-confluence-pages-with-the-rest-api-for-inactive-page-owners-12238


# Scan Confluence pages with the REST API for inactive page owners

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Scan Confluence pages with the REST API for inactive page owners  
**Workflow name (JSON):** Scan Confluence pages for inactive page owners

**Purpose:**  
This workflow scans selected Confluence spaces, retrieves pages via Confluence REST API v2, resolves page owner accounts in bulk, then outputs a report of pages whose owners are **not active** (e.g., deactivated users).

**Primary use cases:**
- Governance/audit: identify pages owned by deactivated or inactive accounts.
- Cleanup: reassign page ownership or update page responsibility.
- Reporting: export a list for admins (email/Slack/CSV can be added after the final aggregation).

### Logical Blocks
1.1 **Manual Start & Configuration**  
1.2 **Retrieve Spaces and Extract Space IDs**  
1.3 **Fetch Pages for the Selected Spaces**  
1.4 **Derive Unique Owner IDs & Bulk-Resolve Users**  
1.5 **Split Pages & Users into Items, Merge on Owner ID**  
1.6 **Filter Pages with Inactive Owners & Build Final Report Output**

---

## 2. Block-by-Block Analysis

### 1.1 Manual Start & Configuration
**Overview:** Starts the workflow manually and defines the Atlassian tenant domain plus the target Confluence space keys.

**Nodes involved:**
- When clicking ‘Execute workflow’
- Set Variables

#### Node: When clicking ‘Execute workflow’
- **Type / role:** Manual Trigger (`manualTrigger`) — entry point.
- **Config choices:** No parameters; execution begins on demand.
- **I/O connections:**  
  - Output → **Set Variables**
- **Edge cases:** None (manual execution only).
- **Version notes:** TypeVersion 1.

#### Node: Set Variables
- **Type / role:** Set (`set`) — provides required configuration values.
- **Configuration (interpreted):**
  - `atlassianDomain`: `https://yourDomain.atlassian.net` (base URL used in all API calls)
  - `spaceKeys`: `space1, space2` (comma-separated list, passed to Confluence “Get spaces”)
- **Key expressions/variables:** Values are literals here; later nodes reference `$json.atlassianDomain` and `$json.spaceKeys`.
- **I/O connections:**  
  - Input ← Manual Trigger  
  - Output → **Confluence - Get Spaces**
- **Edge cases / failures:**
  - Incorrect domain format (missing `https://` or wrong tenant) causes HTTP failures later.
  - `spaceKeys` formatting: Confluence expects specific syntax; spaces after commas may or may not be tolerated.
- **Version notes:** TypeVersion 3.4.

---

### 1.2 Retrieve Spaces and Extract Space IDs
**Overview:** Calls Confluence REST API v2 to fetch spaces matching the configured keys and converts the response into a simple array of `spaceIds`.

**Nodes involved:**
- Confluence - Get Spaces
- Format Space Ids

#### Node: Confluence - Get Spaces
- **Type / role:** HTTP Request (`httpRequest`) — GET `/wiki/api/v2/spaces`.
- **Configuration (interpreted):**
  - **URL:** `{{$json.atlassianDomain}}/wiki/api/v2/spaces`
  - **Query parameters:**
    - `keys`: `{{$json.spaceKeys}}`
    - `type`: `global` (note: JSON shows `=global`; effectively intended as `global`)
  - **Auth:** HTTP Basic Auth via n8n credential “Atlassian Alex” (email + Atlassian API token).
- **I/O connections:**  
  - Input ← Set Variables  
  - Output → **Format Space Ids**
- **Edge cases / failures:**
  - 401/403 if API token invalid or user lacks space read permissions.
  - If `spaceKeys` includes invalid keys, response may return fewer results than expected.
  - Pagination: API may paginate; this node does not handle `next` links.
- **Version notes:** TypeVersion 4.3.

#### Node: Format Space Ids
- **Type / role:** Set (`set`) — transforms spaces list into IDs array.
- **Configuration (interpreted):**
  - Creates `spaceIds` (array) using:  
    `{{$json.results.map(item => item.id)}}`
- **I/O connections:**  
  - Input ← Confluence - Get Spaces  
  - Output → **Confluence - Get Pages**
- **Edge cases / failures:**
  - If response has no `results`, expression fails or returns `[]`.
- **Version notes:** TypeVersion 3.4.

---

### 1.3 Fetch Pages for the Selected Spaces
**Overview:** Fetches pages from the listed space IDs using Confluence REST API v2 and then splits into two coordinated paths: pages stream and owner-IDs stream.

**Nodes involved:**
- Confluence - Get Pages
- Split Out Pages
- Format unique ownerIds

#### Node: Confluence - Get Pages
- **Type / role:** HTTP Request (`httpRequest`) — GET `/wiki/api/v2/pages`.
- **Configuration (interpreted):**
  - **URL:** `{{ $('Set Variables').item.json.atlassianDomain }}/wiki/api/v2/pages`
  - **Query parameters:**
    - `space-id`: `{{$json.spaceIds.join(", ")}}` (comma-separated space IDs)
    - `limit`: `50` (fetch up to 50 pages per request)
  - **Auth:** Same HTTP Basic Auth credential.
- **I/O connections:**
  - Input ← Format Space Ids
  - Output → **Format unique ownerIds** and **Split Out Pages** (fan-out)
- **Edge cases / failures:**
  - Pagination not handled: only first `limit` pages returned (per space-id set).
  - `spaceIds.join(", ")` with spaces might not match API expectations; safest is `","` with no spaces.
  - 400 if `space-id` formatting is rejected.
- **Version notes:** TypeVersion 4.3.

#### Node: Split Out Pages
- **Type / role:** Split Out (`splitOut`) — converts `results[]` into one item per page.
- **Configuration (interpreted):**
  - Field to split: `results`
- **I/O connections:**
  - Input ← Confluence - Get Pages
  - Output → **Merge** (as input 2, index 1)
- **Edge cases / failures:**
  - If `results` is missing or not an array, node yields no items.
- **Version notes:** TypeVersion 1.

#### Node: Format unique ownerIds
- **Type / role:** Set (`set`) — extracts unique owner IDs from pages.
- **Configuration (interpreted):**
  - Creates `ownerIds` using:  
    `{{ [...new Set($json.results.map(item => item.ownerId))] }}`
  - **Important:** In the JSON this field is typed as **string**, but the expression returns an **array**. In practice, this can cause type inconsistencies downstream. The Bulk User Lookup expects an array.
- **I/O connections:**
  - Input ← Confluence - Get Pages
  - Output → **Confluence - Bulk User Lookup**
- **Edge cases / failures:**
  - Pages with missing `ownerId` will include `undefined` in the set unless filtered.
  - Type mismatch (string vs array) can break the POST body formatting.
- **Version notes:** TypeVersion 3.4.

---

### 1.4 Derive Unique Owner IDs & Bulk-Resolve Users
**Overview:** Resolves the collected owner account IDs to user records (including `accountStatus` and `email`) using Confluence REST API v2 bulk endpoint.

**Nodes involved:**
- Confluence - Bulk User Lookup
- Split Out Users
- Filter Inactive Owners

#### Node: Confluence - Bulk User Lookup
- **Type / role:** HTTP Request (`httpRequest`) — POST `/wiki/api/v2/users-bulk`.
- **Configuration (interpreted):**
  - **URL:** `{{ $('Set Variables').item.json.atlassianDomain }}/wiki/api/v2/users-bulk`
  - **Method:** POST
  - **JSON body:**
    ```js
    {
      "accountIds": {{ $json.ownerIds }}
    }
    ```
  - **Auth:** HTTP Basic Auth credential.
- **I/O connections:**
  - Input ← Format unique ownerIds
  - Output → **Split Out Users**
- **Edge cases / failures:**
  - If `ownerIds` is not a valid JSON array, request fails (400).
  - Atlassian may limit bulk sizes; very large spaces could require chunking.
  - 401/403 if user lacks permission to read users.
- **Version notes:** TypeVersion 4.3.

#### Node: Split Out Users
- **Type / role:** Split Out (`splitOut`) — one item per user in response.
- **Configuration:** Field `results`
- **I/O connections:**
  - Input ← Confluence - Bulk User Lookup
  - Output → **Filter Inactive Owners**
- **Edge cases:** No `results` → no user items to merge later.
- **Version notes:** TypeVersion 1.

#### Node: Filter Inactive Owners
- **Type / role:** Filter (`filter`) — keeps only non-active users.
- **Configuration (interpreted):**
  - Condition: `{{$json.accountStatus}}` **notEquals** `active`
- **I/O connections:**
  - Input ← Split Out Users
  - Output → **Merge** (as input 1, index 0)
- **Edge cases / failures:**
  - If `accountStatus` missing, comparison may treat as inactive and pass through.
- **Version notes:** TypeVersion 2.3.

---

### 1.5 Split Pages & Users into Items, Merge on Owner ID
**Overview:** Joins page items with *inactive-owner* user items, enriching page data with user fields where page.ownerId matches user.accountId.

**Nodes involved:**
- Merge

#### Node: Merge
- **Type / role:** Merge (`merge`) — combines two streams by matching IDs.
- **Configuration (interpreted):**
  - Mode: **combine**
  - Join mode: **enrichInput2**  
    → enriches **Input 2** (pages) with fields from **Input 1** (inactive users) when matched.
  - Merge key mapping:
    - Input 1 field: `accountId`
    - Input 2 field: `ownerId`
- **I/O connections:**
  - Input 1 ← Filter Inactive Owners (inactive users)
  - Input 2 ← Split Out Pages (pages)
  - Output → **Filter Inactive Pages**
- **Edge cases / failures:**
  - If page items don’t contain `ownerId`, no matches occur.
  - If user items don’t contain `accountId`, join fails.
  - If the merge outputs unmatched items depends on merge configuration; with enrich mode it typically outputs items from Input 2, enriched when match exists (verify behavior in your n8n version).
- **Version notes:** TypeVersion 3.2.

---

### 1.6 Filter Pages with Inactive Owners & Build Final Report Output
**Overview:** Filters the merged results to keep only pages whose joined owner status indicates inactivity, formats report fields, and aggregates into a single output array.

**Nodes involved:**
- Filter Inactive Pages
- Set Report Data
- Aggregate

#### Node: Filter Inactive Pages
- **Type / role:** Filter (`filter`) — intended to keep pages with inactive owners, but currently configured differently.
- **Configuration (interpreted):**
  - Condition: `{{$json.accountStatus}}` **equals** `active`
  - **alwaysOutputData:** true (so workflow continues even if no items match)
- **Important logic note:**  
  Given earlier steps merge pages with **inactive users**, this filter most likely should be `accountStatus !== active`. As configured, it keeps active owners, which contradicts the sticky note “Filter pages with inactive owner” and the workflow goal.
- **I/O connections:**
  - Input ← Merge
  - Output → **Set Report Data**
- **Edge cases / failures:**
  - If merge does not attach `accountStatus` unless matched, pages without matched inactive user may have `accountStatus` undefined and be filtered out (depending on filter).
- **Version notes:** TypeVersion 2.3.

#### Node: Set Report Data
- **Type / role:** Set (`set`) — builds the final per-page report fields.
- **Configuration (interpreted):**
  - `title`: `{{$json.title}}`
  - `ownerStatus`: `{{$json.accountStatus}}`
  - `ownerEmail`: `{{$json.email}}`
  - `lastUpdated`: `{{$json.version.createdAt}}`
  - `url`: `{{ $('Set Variables').first().json.atlassianDomain }}/wiki{{ $json._links.webui }}`
- **I/O connections:**
  - Input ← Filter Inactive Pages
  - Output → Aggregate
- **Edge cases / failures:**
  - Some page objects may not include `version.createdAt` depending on API expansion; could be undefined.
  - `_links.webui` missing → broken URL expression.
- **Version notes:** TypeVersion 3.4.

#### Node: Aggregate
- **Type / role:** Aggregate (`aggregate`) — gathers all report items into one item containing an array.
- **Configuration (interpreted):**
  - Aggregate mode: “aggregate all item data”
  - Output field: `pagesWithInactiveOwner`
- **I/O connections:**
  - Input ← Set Report Data
  - Output → (none; final result)
- **Edge cases:** Large result sets could increase memory usage.
- **Version notes:** TypeVersion 1.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | Manual Trigger | Manual entry point | — | Set Variables |  |
| Set Variables | Set | Define Atlassian domain + target space keys | When clicking ‘Execute workflow’ | Confluence - Get Spaces | ## How it works… (Setup steps include configuring this node and credentials) |
| Confluence - Get Spaces | HTTP Request | Fetch spaces by keys (Confluence API v2) | Set Variables | Format Space Ids | ## Confluence API Reference (links to V2 endpoints) / ## Get Confluence pages |
| Format Space Ids | Set | Extract `spaceIds[]` from spaces response | Confluence - Get Spaces | Confluence - Get Pages | ## Get Confluence pages |
| Confluence - Get Pages | HTTP Request | Fetch pages for specified space IDs | Format Space Ids | Format unique ownerIds; Split Out Pages | ## Confluence API Reference (links) / ## Get Confluence pages |
| Format unique ownerIds | Set | Compute unique `ownerIds` from pages | Confluence - Get Pages | Confluence - Bulk User Lookup | ## Find inactive users |
| Confluence - Bulk User Lookup | HTTP Request | Bulk resolve users by accountIds | Format unique ownerIds | Split Out Users | ## Confluence API Reference (links) / ## Find inactive users |
| Split Out Users | Split Out | One item per user | Confluence - Bulk User Lookup | Filter Inactive Owners | ## Find inactive users |
| Filter Inactive Owners | Filter | Keep users where `accountStatus !== active` | Split Out Users | Merge | ## Find inactive users |
| Split Out Pages | Split Out | One item per page | Confluence - Get Pages | Merge | ## Get Confluence pages |
| Merge | Merge | Enrich pages with inactive-owner user data via ID join | Filter Inactive Owners; Split Out Pages | Filter Inactive Pages |  |
| Filter Inactive Pages | Filter | Filters merged results (currently equals active) | Merge | Set Report Data | ## Filter pages with inactive owner |
| Set Report Data | Set | Build report fields (title/status/email/URL/date) | Filter Inactive Pages | Aggregate | ## Filter pages with inactive owner |
| Aggregate | Aggregate | Aggregate all report items into `pagesWithInactiveOwner` array | Set Report Data | — | ## Filter pages with inactive owner |
| Sticky Note2 | Sticky Note | Documentation: API references | — | — | ## Confluence API Reference (links to V2 endpoints) |
| Sticky Note | Sticky Note | Label/group: find inactive users | — | — | ## Find inactive users |
| Sticky Note6 | Sticky Note | Notes: permissions, definition, limit, contact | — | — | ### Notes… (includes **office@sus-tech.at**) |
| Sticky Note1 | Sticky Note | Label/group: get pages | — | — | ## Get Confluence pages |
| Sticky Note7 | Sticky Note | Label/group: filter pages with inactive owner | — | — | ## Filter pages with inactive owner |
| Sticky Note8 | Sticky Note | Full explanation + setup steps | — | — | ## How it works / ## Setup steps (includes optional extensions) |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name it: **Scan Confluence pages for inactive page owners**
- Keep it inactive initially (optional).

2) **Add Manual Trigger**
- Node: **Manual Trigger**
- Name: **When clicking ‘Execute workflow’**

3) **Add Set node for configuration**
- Node: **Set**
- Name: **Set Variables**
- Add fields:
  - `atlassianDomain` (String): `https://<yourTenant>.atlassian.net`
  - `spaceKeys` (String): e.g. `ENG,HR` (comma-separated)
- Connect: Manual Trigger → Set Variables

4) **Create Atlassian HTTP Basic Auth credential**
- In n8n Credentials: **HTTP Basic Auth**
  - Username: your Atlassian email
  - Password: your Atlassian API token (from Atlassian account security)
- This credential must be usable for Confluence Cloud API calls.

5) **Add HTTP Request: Get Spaces**
- Node: **HTTP Request**
- Name: **Confluence - Get Spaces**
- Method: GET
- URL: `={{ $json.atlassianDomain }}/wiki/api/v2/spaces`
- Authentication: **Generic Credential Type → HTTP Basic Auth** (select your credential)
- Send Query Params: enabled
- Query params:
  - `keys` = `={{ $json.spaceKeys }}`
  - `type` = `global`
- Connect: Set Variables → Confluence - Get Spaces

6) **Add Set node: extract Space IDs**
- Node: **Set**
- Name: **Format Space Ids**
- Add field:
  - `spaceIds` (Array): `={{ $json.results.map(item => item.id) }}`
- Connect: Confluence - Get Spaces → Format Space Ids

7) **Add HTTP Request: Get Pages**
- Node: **HTTP Request**
- Name: **Confluence - Get Pages**
- Method: GET
- URL: `={{ $('Set Variables').item.json.atlassianDomain }}/wiki/api/v2/pages`
- Authentication: HTTP Basic Auth credential
- Query params:
  - `space-id` = `={{ $json.spaceIds.join(",") }}` (recommended no spaces)
  - `limit` = `50`
- Connect: Format Space Ids → Confluence - Get Pages

8) **Add Split Out: Pages**
- Node: **Split Out**
- Name: **Split Out Pages**
- Field to split out: `results`
- Connect: Confluence - Get Pages → Split Out Pages

9) **Add Set: unique owner IDs**
- Node: **Set**
- Name: **Format unique ownerIds**
- Add field:
  - `ownerIds` (Array) with expression:  
    `={{ [...new Set($json.results.map(item => item.ownerId).filter(Boolean))] }}`
  - (This avoids undefined owner IDs and ensures correct type.)
- Connect: Confluence - Get Pages → Format unique ownerIds

10) **Add HTTP Request: Bulk user lookup**
- Node: **HTTP Request**
- Name: **Confluence - Bulk User Lookup**
- Method: POST
- URL: `={{ $('Set Variables').item.json.atlassianDomain }}/wiki/api/v2/users-bulk`
- Authentication: HTTP Basic Auth credential
- Body type: JSON
- Body:
  - `accountIds`: `={{ $json.ownerIds }}`
- Connect: Format unique ownerIds → Confluence - Bulk User Lookup

11) **Add Split Out: Users**
- Node: **Split Out**
- Name: **Split Out Users**
- Field to split out: `results`
- Connect: Confluence - Bulk User Lookup → Split Out Users

12) **Add Filter: inactive owners**
- Node: **Filter**
- Name: **Filter Inactive Owners**
- Condition:
  - Left: `={{ $json.accountStatus }}`
  - Operation: **not equals**
  - Right: `active`
- Connect: Split Out Users → Filter Inactive Owners

13) **Add Merge: enrich pages with inactive user data**
- Node: **Merge**
- Name: **Merge**
- Mode: **Combine**
- Join / matching:
  - Enable advanced / merge by fields
  - Join mode: **Enrich Input 2**
  - Match:
    - Input 1 field: `accountId` (from user)
    - Input 2 field: `ownerId` (from page)
- Connect:
  - Filter Inactive Owners → Merge (Input 1)
  - Split Out Pages → Merge (Input 2)

14) **Add Filter: keep pages with inactive owners**
- Node: **Filter**
- Name: **Filter Inactive Pages**
- Recommended condition (to match workflow intent):
  - `={{ $json.accountStatus }}` **not equals** `active`
- Optionally keep **Always Output Data** enabled.
- Connect: Merge → Filter Inactive Pages

15) **Add Set: report fields**
- Node: **Set**
- Name: **Set Report Data**
- Fields:
  - `title` = `={{ $json.title }}`
  - `ownerStatus` = `={{ $json.accountStatus }}`
  - `ownerEmail` = `={{ $json.email }}`
  - `lastUpdated` = `={{ $json.version.createdAt }}`
  - `url` = `={{ $('Set Variables').first().json.atlassianDomain }}/wiki{{ $json._links.webui }}`
- Connect: Filter Inactive Pages → Set Report Data

16) **Add Aggregate: final output**
- Node: **Aggregate**
- Name: **Aggregate**
- Mode: **Aggregate All Item Data**
- Destination field name: `pagesWithInactiveOwner`
- Connect: Set Report Data → Aggregate

17) **(Optional) Add output integrations**
- Add Slack/Email/Google Sheets/CSV nodes using `{{$json.pagesWithInactiveOwner}}` from the Aggregate node.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Confluence API Reference: V2 Get Spaces | https://developer.atlassian.com/cloud/confluence/rest/v2/api-group-space/#api-spaces-get |
| Confluence API Reference: V2 Get Pages | https://developer.atlassian.com/cloud/confluence/rest/v2/api-group-page/#api-pages-get |
| Confluence API Reference: V2 Create bulk user lookup using id | https://developer.atlassian.com/cloud/confluence/rest/v2/api-group-user/#api-users-bulk-post |
| Requires Confluence permissions to read spaces, pages and users. | Execution prerequisites |
| Pages are considered problematic if `accountStatus !== active`. | Intended logic definition (note: current workflow filter should be checked) |
| Current page fetch limit is `50` items per request. | Pagination/coverage limitation |
| Need help? ✉️ **office@sus-tech.at** | Support contact (from sticky note) |
| Setup summary: configure `atlassianDomain`, `spaceKeys`, create HTTP Basic Auth (email + API token), assign to all Confluence HTTP nodes; optionally export via email/Slack/CSV using `pagesWithInactiveOwner`. | From “How it works / Setup steps” sticky note |


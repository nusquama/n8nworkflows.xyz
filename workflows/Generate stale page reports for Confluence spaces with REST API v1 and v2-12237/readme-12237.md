Generate stale page reports for Confluence spaces with REST API v1 and v2

https://n8nworkflows.xyz/workflows/generate-stale-page-reports-for-confluence-spaces-with-rest-api-v1-and-v2-12237


# Generate stale page reports for Confluence spaces with REST API v1 and v2

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** This workflow generates a consolidated report of **stale (outdated) Confluence pages** for one or more Confluence Cloud spaces, based on a cutoff age in days. It supports two execution paths:
- **API v2 (recommended):** Fetch spaces → fetch pages → filter pages by last modified date.
- **API v1 (legacy):** Run a **CQL search** that directly returns pages older than the cutoff.

**Primary use cases**
- Auditing and content hygiene (identify pages not updated recently)
- Feeding downstream reporting (CSV export, Slack/email alerts, dashboards)
- Supporting both modern (v2) and legacy (v1/CQL) Confluence API approaches

### Logical blocks
**1.1 Trigger & Configuration**  
Manual trigger and a single “Set Variables” node define all runtime inputs (domain, spaces, cutoff, API choice).

**1.2 API Version Routing**  
A Switch node routes execution to either the API v1 (CQL) branch or the API v2 branch.

**1.3 API v1: CQL Search for Outdated Pages**  
Performs a Confluence REST API v1 search using a CQL query for “lastmodified < now(-Xd)”.

**1.4 API v2: Enumerate Spaces → Pages → Filter by Cutoff**  
Fetches spaces via v2, extracts space IDs, fetches pages via v2, splits items, then filters pages whose `version.createdAt` is older than the cutoff.

**1.5 Normalize Results & Aggregate Report**  
Both branches map results into a unified schema (title/status/lastUpdated/daysOverCutoff/url) and aggregate all items into a single array `stalePages`.

---

## 2. Block-by-Block Analysis

### 2.1 Trigger & Configuration

**Overview:** Starts execution manually and defines shared variables used by all downstream nodes (domain, space keys, cutoff days, API version toggle).

**Nodes involved**
- When clicking ‘Execute workflow’
- Set Variables

#### Node: When clicking ‘Execute workflow’
- **Type / role:** Manual Trigger (`n8n-nodes-base.manualTrigger`) — entry point for interactive runs.
- **Key configuration:** No parameters.
- **Outputs:** One empty starter item to the next node.
- **Failure modes:** None (except workflow not started).

#### Node: Set Variables
- **Type / role:** Set (`n8n-nodes-base.set`) — defines runtime inputs.
- **Configuration choices (interpreted):**
  - `atlassianDomain` (string): Base URL, e.g. `https://yourDomain.atlassian.net`
  - `spaceKeys` (string): Comma-separated keys, e.g. `space1, space2`
  - `cutoffDateDays` (number): e.g. `30`
  - `apiV2` (boolean): `true` uses v2 path, `false` uses v1 path
- **Key expressions/variables:** Values are literal defaults in the node; referenced later via `$('Set Variables')...`.
- **Connections:**  
  - **In:** Manual Trigger  
  - **Out:** Switch API Version
- **Edge cases / failures:**
  - `spaceKeys` is treated as a **string**; later the v1 CQL expression uses `$json.spaceKeys.length`, which measures **string length**, not number of keys.
  - If `atlassianDomain` includes a trailing slash, URLs may become double-slashed (usually tolerated, but not guaranteed).

---

### 2.2 API Version Routing

**Overview:** Selects which Confluence API strategy to use based on the `apiV2` boolean.

**Nodes involved**
- Switch API Version

#### Node: Switch API Version
- **Type / role:** Switch (`n8n-nodes-base.switch`) — conditional branching.
- **Configuration choices:**
  - Output **V1** when `apiV2` is **false**
  - Output **V2** when `apiV2` is **true**
  - Output names are renamed to “V1” and “V2” via `renameOutput`.
- **Key expressions:**
  - `leftValue: {{ $json.apiV2 }}` checked as boolean true/false.
- **Connections:**
  - **In:** Set Variables
  - **Out (V1):** Confluence - Get Outdated Spaces via CQL
  - **Out (V2):** Confluence - Get Spaces
- **Edge cases / failures:**
  - If `apiV2` is missing or not boolean, strict type validation may cause the rule not to match, resulting in no output.

---

### 2.3 API v1: CQL Search for Outdated Pages (Legacy)

**Overview:** Uses Confluence REST API v1 CQL search to retrieve pages older than the cutoff, then normalizes each result for the final report.

**Nodes involved**
- Confluence - Get Outdated Spaces via CQL
- Split Out Pages V1
- Set Report Data V1

#### Node: Confluence - Get Outdated Spaces via CQL
- **Type / role:** HTTP Request (`n8n-nodes-base.httpRequest`) — calls Confluence REST API v1 search endpoint.
- **Endpoint:** `GET {{atlassianDomain}}/wiki/rest/api/search`
- **Auth:** Generic Credential Type → **HTTP Basic Auth** (Atlassian email + API token).
- **Query parameters:**
  - `cql`:
    ```
    lastmodified < now("-{{ cutoffDateDays }}d") AND type=page
    {{ spaceKeys.length > 0 ? `AND space IN (${spaceKeys})` : ``}}
    ```
    Interpreted: only pages older than X days; optionally restrict to spaces.
  - `limit`: `50`
- **Connections:**
  - **In:** Switch API Version (V1)
  - **Out:** Split Out Pages V1
- **Edge cases / failures:**
  - **CQL space restriction formatting risk:** `space IN (...)` in CQL typically expects quoted keys (e.g. `("DOCS","ENG")`). Here it injects the raw `spaceKeys` string (`space1, space2`) which may fail unless formatted exactly as CQL expects.
  - `spaceKeys.length > 0` checks string length, so whitespace-only strings still pass.
  - Pagination: `limit=50` without handling `start`/pagination; large result sets will be truncated.
  - Auth/permissions: missing scope/permissions yields 401/403; restricted spaces/pages won’t appear.
  - Confluence Cloud rate limits or transient 429/5xx.

#### Node: Split Out Pages V1
- **Type / role:** Split Out (`n8n-nodes-base.splitOut`) — turns an array into items.
- **Configuration:** `fieldToSplitOut: results`
- **Connections:**
  - **In:** Confluence - Get Outdated Spaces via CQL
  - **Out:** Set Report Data V1
- **Edge cases / failures:**
  - If the API response has no `results` (error or empty), node may output 0 items.

#### Node: Set Report Data V1
- **Type / role:** Set (`n8n-nodes-base.set`) — maps v1 search results to a normalized schema.
- **Fields created:**
  - `title`: `{{$json.title}}`
  - `status`: `{{$json.content.status}}`
  - `lastUpdated`: `{{$json.lastModified}}`
  - `daysOverCutoff`:
    ```
    Math.max(0,
      Math.floor((Date.now() - Date.parse($json.lastModified)) / 86400000)
      - $('Set Variables').item.json.cutoffDateDays
    )
    ```
  - `url`: `{{ atlassianDomain }}/wiki{{ $json.url }}`
- **Connections:**
  - **In:** Split Out Pages V1
  - **Out:** Aggregate
- **Edge cases / failures:**
  - If `lastModified` is missing or unparsable, `Date.parse()` becomes `NaN` and the math yields `NaN`.
  - `$('Set Variables').item.json.cutoffDateDays` assumes item linkage; in some edge cases (multi-item merges) `.first()` is safer.

---

### 2.4 API v2: Enumerate Spaces → Pages → Filter by Cutoff (Preferred)

**Overview:** Uses Confluence REST API v2 to fetch spaces and pages, then filters pages where the latest version date is older than the cutoff.

**Nodes involved**
- Confluence - Get Spaces
- Format Space Ids
- Confluence - Get Pages
- Split Out Pages V2
- Filter Version by cutoffDate
- Set Report Data V2

#### Node: Confluence - Get Spaces
- **Type / role:** HTTP Request — calls Confluence REST API v2 to list spaces.
- **Endpoint:** `GET {{atlassianDomain}}/wiki/api/v2/spaces`
- **Auth:** HTTP Basic Auth credential.
- **Query parameters:**
  - `keys`: `{{$json.spaceKeys}}` (as provided in Set Variables)
  - `type`: `global`
- **Connections:**
  - **In:** Switch API Version (V2)
  - **Out:** Format Space Ids
- **Edge cases / failures:**
  - If `spaceKeys` formatting doesn’t match what v2 expects (often comma-separated is fine, but depends on API behavior), spaces may be empty.
  - Pagination not handled; default page size may limit results depending on tenant.
  - Permissions: 401/403 or partial visibility.

#### Node: Format Space Ids
- **Type / role:** Set — extracts space IDs from v2 spaces response.
- **Field created:**
  - `spaceIds` (array): `{{$json.results.map(item => item.id)}}`
- **Connections:**
  - **In:** Confluence - Get Spaces
  - **Out:** Confluence - Get Pages
- **Edge cases / failures:**
  - If response schema differs or `results` missing, expression fails.
  - Empty `spaceIds` will lead to an ineffective pages query.

#### Node: Confluence - Get Pages
- **Type / role:** HTTP Request — calls Confluence REST API v2 pages endpoint.
- **Endpoint:** `GET {{atlassianDomain}}/wiki/api/v2/pages`
- **Query parameters:**
  - `space-id`: `{{$json.spaceIds.join(", ")}}`
  - `limit`: `50`
- **Connections:**
  - **In:** Format Space Ids
  - **Out:** Split Out Pages V2
- **Edge cases / failures:**
  - If v2 expects repeated `space-id` params rather than a comma-separated string, this may return fewer/no results (API-dependent).
  - Pagination not handled; only first 50 pages (or first 50 matching) are returned.
  - Permissions and rate limits as above.

#### Node: Split Out Pages V2
- **Type / role:** Split Out — emits one item per page.
- **Configuration:** `fieldToSplitOut: results`
- **Connections:**
  - **In:** Confluence - Get Pages
  - **Out:** Filter Version by cutoffDate
- **Edge cases / failures:**
  - Missing/empty `results` yields 0 items.

#### Node: Filter Version by cutoffDate
- **Type / role:** Filter (`n8n-nodes-base.filter`) — keeps only pages with old `version.createdAt`.
- **Condition:** `{{$json.version.createdAt}} <= {{ now - cutoffDateDays }}`
  - Right side computed as:
    ```
    new Date(Date.now() - $('Set Variables').first().json.cutoffDateDays * 24*60*60*1000)
    ```
- **Connections:**
  - **In:** Split Out Pages V2
  - **Out:** Set Report Data V2 (only matching/true branch)
- **Edge cases / failures:**
  - If `version.createdAt` is missing, strict validation may drop items or error depending on n8n behavior.
  - Timezone: `createdAt` is ISO; comparison is generally safe but can be off by hours near cutoff boundaries.

#### Node: Set Report Data V2
- **Type / role:** Set — maps v2 page items to the same normalized schema.
- **Fields created:**
  - `title`: `{{$json.title}}`
  - `status`: `{{$json.status}}`
  - `lastUpdated`: `{{$json.version.createdAt}}`
  - `daysOverCutoff`: same math as v1 but using `version.createdAt`
  - `url`: `{{atlassianDomain}}/wiki{{ $json._links.webui }}`
- **Connections:**
  - **In:** Filter Version by cutoffDate
  - **Out:** Aggregate
- **Edge cases / failures:**
  - If `_links.webui` missing, url becomes invalid.
  - If `version.createdAt` unparsable, `daysOverCutoff` becomes `NaN`.

---

### 2.5 Normalize Results & Aggregate Report

**Overview:** Collects all stale page items (from either API path) into a single output array `stalePages`.

**Nodes involved**
- Aggregate

#### Node: Aggregate
- **Type / role:** Aggregate (`n8n-nodes-base.aggregate`) — aggregates all incoming items into one item.
- **Configuration:**
  - Mode: `aggregateAllItemData`
  - Destination field: `stalePages`
- **Connections:**
  - **In:** Set Report Data V1 and Set Report Data V2
  - **Out:** (None in provided workflow; final output is this node’s data)
- **Edge cases / failures:**
  - If zero stale pages pass through, `stalePages` may be an empty array (or node may output empty depending on n8n version/settings).
  - If both branches run (they do not here), data would be combined—currently mutually exclusive.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | Manual Trigger | Manual start entry point | — | Set Variables |  |
| Set Variables | Set | Defines domain, space keys, cutoff, API toggle | When clicking ‘Execute workflow’ | Switch API Version |  |
| Switch API Version | Switch | Routes to API v1 or v2 branch | Set Variables | Confluence - Get Outdated Spaces via CQL; Confluence - Get Spaces |  |
| Confluence - Get Outdated Spaces via CQL | HTTP Request | Confluence v1 CQL search for stale pages | Switch API Version | Split Out Pages V1 | ## Confluence API v1 |
| Split Out Pages V1 | Split Out | Split v1 `results` array into items | Confluence - Get Outdated Spaces via CQL | Set Report Data V1 | ## Confluence API v1 |
| Set Report Data V1 | Set | Normalize v1 page fields for report | Split Out Pages V1 | Aggregate | ## Confluence API v1 |
| Confluence - Get Spaces | HTTP Request | Confluence v2 list spaces by keys | Switch API Version | Format Space Ids | ## Confluence API v2 |
| Format Space Ids | Set | Extract `spaceIds` array from spaces results | Confluence - Get Spaces | Confluence - Get Pages | ## Confluence API v2 |
| Confluence - Get Pages | HTTP Request | Confluence v2 list pages for spaces | Format Space Ids | Split Out Pages V2 | ## Confluence API v2 |
| Split Out Pages V2 | Split Out | Split v2 `results` array into items | Confluence - Get Pages | Filter Version by cutoffDate | ## Confluence API v2 |
| Filter Version by cutoffDate | Filter | Keep only pages older than cutoff | Split Out Pages V2 | Set Report Data V2 | ## Confluence API v2 |
| Set Report Data V2 | Set | Normalize v2 page fields for report | Filter Version by cutoffDate | Aggregate | ## Confluence API v2 |
| Aggregate | Aggregate | Combine all stale pages into `stalePages` array | Set Report Data V1; Set Report Data V2 | — |  |
| Sticky Note | Sticky Note | Comment block | — | — | ## Confluence API v2 |
| Sticky Note1 | Sticky Note | Comment block | — | — | ## Confluence API v1 |
| Sticky Note2 | Sticky Note | Comment block | — | — | ## Confluence API Reference\n* [V1 CQL Search](https://developer.atlassian.com/cloud/confluence/rest/v1/api-group-search/#api-wiki-rest-api-search-get)\n* [V2 Get Spaces](https://developer.atlassian.com/cloud/confluence/rest/v2/api-group-space/#api-spaces-get)\n* [V2 Get Pages](https://developer.atlassian.com/cloud/confluence/rest/v2/api-group-page/#api-pages-get) |
| Sticky Note6 | Sticky Note | Comment block | — | — | ### Notes\n- API v2 is preferred and future-proof; v1 exists for legacy compatibility.\n- Pagination limits are set to 50 items per request — increase if needed.\n- If results are empty:\n  - Verify space keys\n  - Confirm cutoff days logic\n  - Check API permissions for the token\n- Tested against Confluence Cloud.\n- Need help? ✉️ **office@sus-tech.at** |
| Sticky Note7 | Sticky Note | Comment block | — | — | ## How it works\nThis workflow identifies stale Confluence pages based on last modification date. A **Set Variables** node defines the core inputs: your Atlassian domain, target space keys, an age threshold in days and whether to use the v1 or v2 Confluence API.\n\n- If **API v1** is selected, the workflow runs a CQL search that directly returns pages older than the cutoff date.\n- If **API v2** is enabled (recommended), the workflow first fetches spaces, then pages within those spaces, and filters them by their last modified timestamp.\n\nRegardless of the path, each page is normalized into a consistent structure (title, space, URL, last updated, author). All results are combined into a single `stalePages` array, making the output easy to reuse for reporting, notifications, or exports.\n\n## Setup steps\n1. Open the **Set Variables** node and configure:\n   - `atlassianDomain`: Your Confluence base URL\n   - `spaceKeys`: Comma-separated space keys (e.g. `DOCS,ENG`)\n   - `cutoffDateDays`: Age threshold in days (e.g. `90`)\n   - `apiV2`: `true` to use API v2, `false` for legacy CQL\n2. Create an **HTTP Basic Auth** credential in n8n using your Atlassian email and API token.\n3. Assign this credential to all HTTP Request nodes in the workflow.\n4. (Optional) Add follow-up nodes for email, Slack alerts, or CSV export using the `stalePages` output. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** named: `Report stale pages in Confluence`.

2. **Add node: Manual Trigger**
   - Node type: **Manual Trigger**
   - Name: `When clicking ‘Execute workflow’`

3. **Add node: Set**
   - Name: `Set Variables`
   - Add fields:
     - `atlassianDomain` (String): `https://yourDomain.atlassian.net`
     - `spaceKeys` (String): `space1, space2`
     - `cutoffDateDays` (Number): `30`
     - `apiV2` (Boolean): `true`
   - Connect: Manual Trigger → Set Variables

4. **Add node: Switch**
   - Name: `Switch API Version`
   - Create two rules (with renamed outputs):
     - Output **V1**: condition `{{$json.apiV2}}` is **false**
     - Output **V2**: condition `{{$json.apiV2}}` is **true**
   - Connect: Set Variables → Switch API Version

5. **Create credential: HTTP Basic Auth (Atlassian)**
   - In n8n Credentials, add **HTTP Basic Auth**
   - Username: your Atlassian account email
   - Password: your Atlassian API token
   - Save as (example): `Atlassian Alex`
   - This credential will be reused on all HTTP Request nodes.

---

### Build the API v1 branch (CQL)

6. **Add node: HTTP Request**
   - Name: `Confluence - Get Outdated Spaces via CQL`
   - Method: GET
   - URL: `={{ $json.atlassianDomain }}/wiki/rest/api/search`
   - Authentication: **Generic Credential Type** → **HTTP Basic Auth** (select your Atlassian credential)
   - Send Query Parameters: enabled
   - Query parameters:
     - `cql`:
       ```
       =lastmodified < now("-{{ $json.cutoffDateDays }}d") AND type=page {{ $json.spaceKeys.length > 0 ? `AND space IN (${$json.spaceKeys})` : ``}}
       ```
     - `limit`: `50`
   - Connect: Switch API Version (V1 output) → this node

7. **Add node: Split Out**
   - Name: `Split Out Pages V1`
   - Field to split out: `results`
   - Connect: CQL node → Split Out Pages V1

8. **Add node: Set**
   - Name: `Set Report Data V1`
   - Add fields (String unless derived):
     - `title`: `={{ $json.title }}`
     - `status`: `={{ $json.content.status }}`
     - `lastUpdated`: `={{ $json.lastModified }}`
     - `daysOverCutoff`:
       `={{ Math.max(0, Math.floor((Date.now() - Date.parse($json.lastModified)) / 86400000) - $('Set Variables').item.json.cutoffDateDays) }}`
     - `url`: `={{ $('Set Variables').first().json.atlassianDomain }}/wiki{{ $json.url }}`
   - Connect: Split Out Pages V1 → Set Report Data V1

---

### Build the API v2 branch (spaces → pages → filter)

9. **Add node: HTTP Request**
   - Name: `Confluence - Get Spaces`
   - Method: GET
   - URL: `={{ $json.atlassianDomain }}/wiki/api/v2/spaces`
   - Authentication: HTTP Basic Auth credential (same as above)
   - Query parameters:
     - `keys`: `={{ $json.spaceKeys }}`
     - `type`: `=global`
   - Connect: Switch API Version (V2 output) → Confluence - Get Spaces

10. **Add node: Set**
    - Name: `Format Space Ids`
    - Field:
      - `spaceIds` (Array):
        `={{ $json.results.map(item => item.id) }}`
    - Connect: Confluence - Get Spaces → Format Space Ids

11. **Add node: HTTP Request**
    - Name: `Confluence - Get Pages`
    - Method: GET
    - URL: `={{ $('Set Variables').item.json.atlassianDomain }}/wiki/api/v2/pages`
    - Authentication: HTTP Basic Auth credential
    - Query parameters:
      - `space-id`: `={{ $json.spaceIds.join(", ") }}`
      - `limit`: `50`
    - Connect: Format Space Ids → Confluence - Get Pages

12. **Add node: Split Out**
    - Name: `Split Out Pages V2`
    - Field to split out: `results`
    - Connect: Confluence - Get Pages → Split Out Pages V2

13. **Add node: Filter**
    - Name: `Filter Version by cutoffDate`
    - Condition (DateTime “before or equals”):
      - Left value: `={{ $json.version.createdAt }}`
      - Right value:
        `={{ new Date(Date.now() - $('Set Variables').first().json.cutoffDateDays * 24 * 60 * 60 * 1000) }}`
    - Connect: Split Out Pages V2 → Filter Version by cutoffDate

14. **Add node: Set**
    - Name: `Set Report Data V2`
    - Fields:
      - `title`: `={{ $json.title }}`
      - `status`: `={{ $json.status }}`
      - `lastUpdated`: `={{ $json.version.createdAt }}`
      - `daysOverCutoff`:
        `={{ Math.max(0, Math.floor((Date.now() - Date.parse($json.version.createdAt)) / 86400000) - $('Set Variables').item.json.cutoffDateDays) }}`
      - `url`: `={{ $('Set Variables').first().json.atlassianDomain }}/wiki{{ $json._links.webui }}`
    - Connect: Filter Version by cutoffDate → Set Report Data V2

---

### Aggregate final output

15. **Add node: Aggregate**
    - Name: `Aggregate`
    - Operation: **Aggregate All Item Data**
    - Destination field name: `stalePages`
    - Connect:
      - Set Report Data V1 → Aggregate
      - Set Report Data V2 → Aggregate

16. **(Optional) Add next steps**
    - Example: add Slack/Email/Spreadsheet nodes after **Aggregate** using `{{$json.stalePages}}`.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| API v2 is preferred and future-proof; v1 exists for legacy compatibility. | Sticky note “Notes” |
| Pagination limits are set to 50 items per request — increase if needed. | Sticky note “Notes” |
| If results are empty: verify space keys; confirm cutoff logic; check API permissions for the token. | Sticky note “Notes” |
| Tested against Confluence Cloud. | Sticky note “Notes” |
| Need help? ✉️ **office@sus-tech.at** | Sticky note “Notes” |
| Confluence API Reference: V1 CQL Search | https://developer.atlassian.com/cloud/confluence/rest/v1/api-group-search/#api-wiki-rest-api-search-get |
| Confluence API Reference: V2 Get Spaces | https://developer.atlassian.com/cloud/confluence/rest/v2/api-group-space/#api-spaces-get |
| Confluence API Reference: V2 Get Pages | https://developer.atlassian.com/cloud/confluence/rest/v2/api-group-page/#api-pages-get |
| Workflow explanation and setup steps are embedded as a large sticky note (“How it works”). | Included in workflow canvas |


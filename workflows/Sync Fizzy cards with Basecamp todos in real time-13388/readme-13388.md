Sync Fizzy cards with Basecamp todos in real time

https://n8nworkflows.xyz/workflows/sync-fizzy-cards-with-basecamp-todos-in-real-time-13388


# Sync Fizzy cards with Basecamp todos in real time

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow listens to **Fizzy** card events (via webhook) and keeps **Basecamp** To-dos synchronized in near real time. It creates a new Basecamp To-do when a Fizzy card is published, and updates an existing Basecamp To-do when the card is assigned/unassigned or its status changes (closed/reopened/postponed/triage).

**Target use cases:**
- Teams using Fizzy for card-based triage and Basecamp for execution tracking.
- Automatic creation of tasks in Basecamp based on Fizzy cards.
- Keeping assignees and completion state consistent across tools.

### Logical blocks
1.1 **Webhook intake & Basecamp bootstrap**  
Receives Fizzy events, injects Basecamp account ID, and fetches Basecamp Projects + People.

1.2 **Mapping Fizzy → Basecamp project/todoset/todolist + assignees**  
Finds the Basecamp project matching the Fizzy board name, locates the project’s todoset, fetches todolists, then selects a todolist from the Fizzy card’s first tag (or a default), and maps assignees by email.

1.3 **Todolist existence check & optional creation**  
If the matching todolist doesn’t exist, it creates it and merges the new todolist ID back into the working payload.

1.4 **Create vs Update routing**  
If the Fizzy action is `card_published`, create a new Basecamp To-do. Otherwise, fetch existing To-dos (active + completed), find the correct one by embedded “Fizzy ID”, and prepare update instructions.

1.5 **Update handling: assignees + completion status**  
For assignment changes, update the To-do’s assignees/details. For status changes, complete/uncomplete the To-do according to the Fizzy action.

---

## 2. Block-by-Block Analysis

### 1.1 Webhook intake & Basecamp bootstrap

**Overview:**  
Accepts incoming Fizzy webhook payloads, adds a configured Basecamp Account ID, then fetches Basecamp projects and people (needed for mapping project and assignees).

**Nodes involved:**
- Receive Fizzy webhook
- Set Basecamp account ID
- Fetch Basecamp projects
- Fetch Basecamp people
- Combine projects and people

#### Node: Receive Fizzy webhook
- **Type / role:** `Webhook` — workflow trigger / entry point.
- **Configuration choices:**
  - Method: `POST`
  - Path: `fizzyxbasecamp` (your n8n base URL determines the full endpoint)
  - Response: `lastNode` (the HTTP response returns the last node output)
- **Inputs/Outputs:** Entry node → outputs to “Set Basecamp account ID”.
- **Edge cases / failures:**
  - Invalid Fizzy webhook format (missing expected fields like `body.board.name`).
  - If the workflow errors downstream, webhook caller may receive an error.
  - Ensure Fizzy is configured to send POST to the correct URL and includes all events.
- **Version notes:** Node typeVersion `2.1`.

#### Node: Set Basecamp account ID
- **Type / role:** `Code` — enrich payload with Basecamp account ID.
- **Configuration choices:**
  - Hardcoded constant: `BASECAMP_ACCOUNT_ID = "<Put your account ID Here>"`
  - Pass-through of webhook JSON and adds `basecampAccountId`.
- **Key variables/expressions:**
  - Uses `$input.first().json` as webhook payload.
- **Inputs/Outputs:**
  - Input: from Webhook
  - Output: to both “Fetch Basecamp projects” and “Fetch Basecamp people” (fan-out)
- **Edge cases / failures:**
  - If account ID is not replaced, subsequent Basecamp URLs will be invalid and requests will fail (404/401).
- **Version notes:** Code node typeVersion `2`.

#### Node: Fetch Basecamp projects
- **Type / role:** `HTTP Request` — lists Basecamp projects.
- **Configuration choices:**
  - URL: `https://3.basecampapi.com/{{ basecampAccountId }}/projects.json`
  - Auth: predefined credential type `basecamp4OAuth2Api`
- **Key expressions:**
  - `{{ $('Set Basecamp account ID').first().json.basecampAccountId }}`
- **Inputs/Outputs:**
  - Input: from “Set Basecamp account ID”
  - Output: to “Combine projects and people” (input 0)
- **Edge cases / failures:**
  - OAuth token expired/invalid → 401.
  - Account ID mismatch → 404.
  - Large orgs: paging not handled here; if Basecamp paginates projects, you may miss matches.
- **Version notes:** HTTP node typeVersion `4.3`.

#### Node: Fetch Basecamp people
- **Type / role:** `HTTP Request` — lists Basecamp people (for assignee matching).
- **Configuration choices:**
  - URL: `https://3.basecampapi.com/{{ basecampAccountId }}/people.json`
  - Auth: `basecamp4OAuth2Api`
- **Inputs/Outputs:**
  - Input: from “Set Basecamp account ID”
  - Output: to “Combine projects and people” (input 1)
- **Edge cases / failures:**
  - Same OAuth/account issues as projects.
  - Large directory: pagination not configured; may not retrieve all people.
- **Version notes:** HTTP node typeVersion `4.3`.

#### Node: Combine projects and people
- **Type / role:** `Merge` — joins both branches so the workflow can proceed with both datasets available.
- **Configuration choices:** Default merge behavior (no explicit mode configured in parameters).
- **Inputs/Outputs:**
  - Inputs: projects (index 0) + people (index 1)
  - Output: to “Match project and get todoset”
- **Edge cases / failures:**
  - If one branch fails, merge will not produce expected combined flow.
- **Version notes:** Merge typeVersion `3`.

---

### 1.2 Mapping Fizzy → Basecamp project/todoset/todolist + assignees

**Overview:**  
Matches the Fizzy board to a Basecamp project by name, extracts the project’s todoset from the dock, fetches todolists, then maps the Fizzy card’s first tag to a todolist title and maps Fizzy assignees to Basecamp people via email.

**Nodes involved:**
- Match project and get todoset
- Fetch project todolists
- Match todolist and assignees

#### Node: Match project and get todoset
- **Type / role:** `Code` — selects correct Basecamp project and todoset; forwards people list.
- **Configuration choices:**
  - Reads Fizzy board name: `webhookData.body.board.name`
  - Reads Fizzy action: `webhookData.body.action`
  - Finds Basecamp project by case-insensitive exact name match
  - Locates dock item with `name === 'todoset'`
- **Key variables:**
  - Outputs: `projectId`, `todosetId`, `boardName`, `fizzyAction`, `webhookData`, `basecampPeople` (mapped to `.json`)
- **Inputs/Outputs:**
  - Uses data from:
    - `Receive Fizzy webhook`
    - `Fetch Basecamp projects` (via `.all()`)
    - `Fetch Basecamp people` (via `.all()`)
  - Output: to “Fetch project todolists”
- **Edge cases / failures:**
  - No matching Basecamp project → throws error.
  - Todos tool not enabled / no todoset in dock → throws error.
  - Fizzy payload missing `body.board.name` → expression error.
- **Version notes:** Code typeVersion `2`.

#### Node: Fetch project todolists
- **Type / role:** `HTTP Request` — retrieves all todolists for the project todoset.
- **Configuration choices:**
  - URL: `/buckets/{projectId}/todosets/{todosetId}/todolists.json`
  - Auth: `basecamp4OAuth2Api`
- **Inputs/Outputs:**
  - Input: from “Match project and get todoset”
  - Output: to “Match todolist and assignees”
- **Edge cases / failures:**
  - 404 if IDs are wrong (project mismatch, missing todoset).
  - Pagination not configured—if many todolists, potential incomplete retrieval (depends on Basecamp API behavior).
- **Version notes:** HTTP typeVersion `4.3`.

#### Node: Match todolist and assignees
- **Type / role:** `Code` — chooses todolist based on Fizzy tag and maps assignees by email.
- **Configuration choices:**
  - Uses first Fizzy tag: `webhookData.body.eventable.tags[0]`
  - Default list name if no tags: `"Fizzy Todo"`
  - Matches todolist by case-insensitive exact equality against todolist title
  - Maps assignee emails (`email_address`) to Basecamp people emails
  - Builds a “baseData” payload used downstream for create/update routing
- **Key outputs:**
  - Always outputs: `projectId`, `todosetId`, `cardTitle`, `cardDescription`, `assigneeIds`, `fizzyAction`, `fizzyCardId`, `basecampAccountId`, `firstTag`, `isDefaultList`
  - If matched todolist: `todolistId`, `todolistFound: true`
  - If not matched: `todolistFound: false`, `newTodolistName`
- **Inputs/Outputs:**
  - Input: todolists list from HTTP node (`$input.all()`)
  - References: “Match project and get todoset” for webhookData + basecampPeople
  - Output: to “Check if todolist exists”
- **Edge cases / failures:**
  - If Fizzy assignee email doesn’t exist in Basecamp people, that assignee is skipped (silent).
  - If Fizzy tag differs slightly from todolist title (spacing/punctuation), it won’t match and will create a new todolist.
  - Missing `eventable` structure in webhook breaks.
- **Version notes:** Code typeVersion `2`.

---

### 1.3 Todolist existence check & optional creation

**Overview:**  
Routes depending on whether the desired Basecamp todolist exists. If missing, creates it and injects the new todolist ID back into the main data flow.

**Nodes involved:**
- Check if todolist exists
- Create new todolist
- Add new todolist ID

#### Node: Check if todolist exists
- **Type / role:** `IF` — boolean check on `todolistFound`.
- **Configuration choices:**
  - Condition: `{{ $json.todolistFound }}` is `true`
- **Inputs/Outputs:**
  - True output → “Check if New Card”
  - False output → “Create new todolist”
- **Edge cases / failures:**
  - If `todolistFound` is missing or not boolean, strict validation may fail.
- **Version notes:** IF typeVersion `2`.

#### Node: Create new todolist
- **Type / role:** `Basecamp (community) node` — creates a todolist in the project’s todoset.
- **Configuration choices:**
  - Resource: `todolist`
  - Operation: `createTodolist`
  - `bucketId` = projectId
  - `todosetId` = todosetId
  - Name = `newTodolistName`
  - Description: “Auto-created from Fizzy”
- **Credentials:** `basecamp4OAuth2Api` (OAuth2)
- **Inputs/Outputs:**
  - Input: from IF (false branch)
  - Output: to “Add new todolist ID”
- **Edge cases / failures:**
  - Permission issues on project → 403.
  - Duplicate todolist names allowed by Basecamp; you may create duplicates if tag naming varies.
- **Version notes:** Basecamp node typeVersion `1`. Requires installing `n8n-nodes-basecamp` (community node).

#### Node: Add new todolist ID
- **Type / role:** `Code` — merges created todolist ID into original payload.
- **Configuration choices:**
  - Takes created todolist `id` from `$input.first().json`
  - Pulls original data from `$('Match todolist and assignees').first().json`
  - Outputs combined object with `todolistId` and `todolistFound: true`
- **Inputs/Outputs:**
  - Input: from “Create new todolist”
  - Output: to “Check if New Card”
- **Edge cases / failures:**
  - If Basecamp node returns a different structure (no `id`), next steps fail.
- **Version notes:** Code typeVersion `2`.

---

### 1.4 Create vs Update routing

**Overview:**  
If the event is a newly published card, create a new Basecamp To-do. Otherwise, fetch all active and completed todos, merge them, locate the matching todo using the embedded “Fizzy ID” marker, and prepare update instructions.

**Nodes involved:**
- Check if New Card
- Create new todo
- Fetch active todos
- Fetch completed todos
- Combine all todos
- Find todo and prepare update

#### Node: Check if New Card
- **Type / role:** `IF` — routes “create” vs “update”.
- **Configuration choices:**
  - Condition: `{{ $json.fizzyAction }}` equals `card_published`
- **Inputs/Outputs:**
  - True → “Create new todo”
  - False → in parallel: “Fetch active todos” and “Fetch completed todos”
- **Edge cases / failures:**
  - If Fizzy action names change or other “new card” actions exist, they won’t create.
- **Version notes:** IF typeVersion `2`.

#### Node: Create new todo
- **Type / role:** `Basecamp (community) node` — creates a Basecamp todo.
- **Configuration choices:**
  - Resource: `todo`
  - Operation: `createTodo`
  - `bucketId` = projectId
  - `todolistId` = todolistId
  - Content = card title
  - Assignees = `assigneeIds`
  - Notify = true
  - Description template embeds the mapping key:
    - `Fizzy ID: {{$json.fizzyCardId}} - Please Don't Delete!` then the card description
- **Inputs/Outputs:** Input from IF(true). (No downstream connections.)
- **Edge cases / failures:**
  - If users delete/alter the “Fizzy ID:” line in Basecamp, later updates cannot find the todo.
  - If assigneeIds array empty, todo is created unassigned (usually acceptable).
- **Version notes:** Basecamp node typeVersion `1` (community node).

#### Node: Fetch active todos
- **Type / role:** `HTTP Request` — fetches todos for the todolist (active).
- **Configuration choices:**
  - URL: `/buckets/{projectId}/todolists/{todolistId}/todos.json`
  - Pagination: uses the HTTP `Link` header and extracts `rel="next"` URL with regex.
- **Inputs/Outputs:**
  - Input: from IF(false)
  - Output: to “Combine all todos” (input 0)
- **Edge cases / failures:**
  - If Basecamp changes link header format, nextURL regex may fail → incomplete paging.
  - Large lists: many pages; timeouts possible.
- **Version notes:** HTTP typeVersion `4.4`.

#### Node: Fetch completed todos
- **Type / role:** `HTTP Request` — fetches completed todos.
- **Configuration choices:**
  - URL: `/todos.json?completed=true`
  - Same Link-header pagination as above.
  - `sendQuery: true` but query parameters list appears empty; `completed=true` is already in URL.
- **Inputs/Outputs:**
  - Input: from IF(false)
  - Output: to “Combine all todos” (input 1)
- **Edge cases / failures:**
  - Same pagination/timeouts as active.
  - Potential duplication if API returns completed in both endpoints (unlikely, but watch).
- **Version notes:** HTTP typeVersion `4.4`.

#### Node: Combine all todos
- **Type / role:** `Merge` — combines active and completed lists into one stream.
- **Configuration choices:** Default merge settings.
- **Inputs/Outputs:**
  - Inputs: active (0) + completed (1)
  - Output: to “Find todo and prepare update”
- **Edge cases / failures:**
  - If one branch returns 0 items and the other returns many, default merge behavior can be confusing depending on merge mode. Here it works because the next Code node uses `$input.all()` to collect everything it receives.
- **Version notes:** Merge typeVersion `3`.

#### Node: Find todo and prepare update
- **Type / role:** `Code` — locates matching Basecamp todo and builds an update plan.
- **Configuration choices:**
  - Reads route data from `$('Check if New Card').first().json` (this is the “context” object that includes fizzyAction, ids, etc.)
  - Searches among all merged todos for a description containing `Fizzy ID:\s*{fizzyCardId}` (case-insensitive regex)
  - Throws a detailed error listing available todos if none found.
  - Sets flags:
    - `shouldComplete` for `card_closed`
    - `shouldReopen` for `card_reopened`, `card_postponed`, `card_auto_postponed`, `card_sent_back_to_triage`
- **Outputs:**
  - `todoId`, `projectId`, `fizzyAction`, `assigneeIds`, `cardTitle`, `cardDescription`, `fizzyCardId`, current todo state, and status flags.
- **Inputs/Outputs:**
  - Input: all todos from merge (`$input.all()`)
  - Output: to “Check if assignment change”
- **Edge cases / failures:**
  - If description is empty or edited to remove “Fizzy ID”, matching fails and throws.
  - If multiple todos include the same Fizzy ID (duplicates), it picks the first match.
  - Depends on `Check if New Card` output being available; in complex execution paths, relying on `$('Node').first()` can be brittle.
- **Version notes:** Code typeVersion `2`.

---

### 1.5 Update handling: assignees + completion status

**Overview:**  
Determines whether the Fizzy event is an assignment change and updates the Basecamp todo accordingly; then, for close/reopen/postpone-type actions, it completes or uncompletes the todo.

**Nodes involved:**
- Check if assignment change
- Update todo assignees
- Check if card closed
- Mark todo as complete
- Check if card reopened
- Mark todo as incomplete

#### Node: Check if assignment change
- **Type / role:** `IF` — detects assignment-related events.
- **Configuration choices:**
  - OR condition: `fizzyAction == card_assigned` OR `fizzyAction == card_unassigned`
- **Inputs/Outputs:**
  - True → “Update todo assignees”
  - False → “Check if card closed”
- **Edge cases / failures:**
  - If Fizzy uses different action names for assignment changes, updates won’t run.
- **Version notes:** IF typeVersion `2`.

#### Node: Update todo assignees
- **Type / role:** `Basecamp (community) node` — updates todo fields.
- **Configuration choices:**
  - Operation: `updateTodo`
  - Updates:
    - `notify: true`
    - `assigneeIds` from payload
    - `description` re-written with the “Fizzy ID” header + card description
    - `content` also set to cardTitle (keeps title in sync)
- **Inputs/Outputs:** Input from assignment IF(true). No downstream.
- **Edge cases / failures:**
  - This rewrites description; manual notes in Basecamp description may be overwritten.
  - If Basecamp API requires additional fields for update in some scenarios, may fail.
- **Version notes:** Basecamp node typeVersion `1`.

#### Node: Check if card closed
- **Type / role:** `IF` — checks for completion.
- **Configuration choices:** `fizzyAction == card_closed`
- **Inputs/Outputs:**
  - True → “Mark todo as complete”
  - False → “Check if card reopened”
- **Edge cases / failures:** Action naming mismatch.
- **Version notes:** IF typeVersion `2`.

#### Node: Mark todo as complete
- **Type / role:** `Basecamp (community) node` — completes a todo.
- **Configuration choices:** Operation `completeTodo`
- **Inputs/Outputs:** From “Check if card closed”(true). No downstream.
- **Edge cases / failures:** If todo already completed, Basecamp may return a no-op or error depending on API behavior.
- **Version notes:** Basecamp node typeVersion `1`.

#### Node: Check if card reopened
- **Type / role:** `IF` — checks for actions that should “reopen/uncomplete”.
- **Configuration choices:** OR across:
  - `card_reopened`
  - `card_postponed`
  - `card_auto_postponed`
  - `card_sent_back_to_triage`
- **Inputs/Outputs:** True → “Mark todo as incomplete”. False → end.
- **Edge cases / failures:** If additional Fizzy actions should uncomplete, they are not covered.
- **Version notes:** IF typeVersion `2`.

#### Node: Mark todo as incomplete
- **Type / role:** `Basecamp (community) node` — uncompletes a todo.
- **Configuration choices:** Operation `uncompleteTodo`
- **Inputs/Outputs:** From “Check if card reopened”(true). No downstream.
- **Edge cases / failures:** If todo is already incomplete, may be no-op or error.
- **Version notes:** Basecamp node typeVersion `1`.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Receive Fizzy webhook | Webhook | Entry point receiving Fizzy events | — | Set Basecamp account ID | Receives webhook from Fizzy and fetches Basecamp data |
| Set Basecamp account ID | Code | Inject Basecamp account ID into payload | Receive Fizzy webhook | Fetch Basecamp projects; Fetch Basecamp people | Receives webhook from Fizzy and fetches Basecamp data |
| Fetch Basecamp projects | HTTP Request | Retrieve Basecamp projects list | Set Basecamp account ID | Combine projects and people | Receives webhook from Fizzy and fetches Basecamp data |
| Fetch Basecamp people | HTTP Request | Retrieve Basecamp people list | Set Basecamp account ID | Combine projects and people | Receives webhook from Fizzy and fetches Basecamp data |
| Combine projects and people | Merge | Join project + people branches | Fetch Basecamp projects; Fetch Basecamp people | Match project and get todoset | Receives webhook from Fizzy and fetches Basecamp data |
| Match project and get todoset | Code | Match Fizzy board to Basecamp project and get todoset | Combine projects and people | Fetch project todolists | Matches Fizzy Data to Basecamp Data (Projects, Todoset, Todolist) |
| Fetch project todolists | HTTP Request | List todolists in the todoset | Match project and get todoset | Match todolist and assignees | Matches Fizzy Data to Basecamp Data (Projects, Todoset, Todolist) |
| Match todolist and assignees | Code | Select/create todolist name; map assignees by email | Fetch project todolists | Check if todolist exists | Matches Fizzy Data to Basecamp Data (Projects, Todoset, Todolist) |
| Check if todolist exists | IF | Route based on todolistFound | Match todolist and assignees | Check if New Card; Create new todolist | Matches Fizzy Data to Basecamp Data (Projects, Todoset, Todolist) |
| Create new todolist | Basecamp (community) | Create missing todolist | Check if todolist exists (false) | Add new todolist ID | Matches Fizzy Data to Basecamp Data (Projects, Todoset, Todolist) |
| Add new todolist ID | Code | Merge created todolistId back into payload | Create new todolist | Check if New Card | Matches Fizzy Data to Basecamp Data (Projects, Todoset, Todolist) |
| Check if New Card | IF | Create vs update path | Check if todolist exists (true); Add new todolist ID | Create new todo; Fetch active todos; Fetch completed todos | Creates new todo when fizzy card status is equal to published (New) / Updates existing todo when card changes |
| Create new todo | Basecamp (community) | Create Basecamp todo for new card | Check if New Card (true) | — | Creates new todo when fizzy card status is equal to published (New) |
| Fetch active todos | HTTP Request | Retrieve active todos for todolist | Check if New Card (false) | Combine all todos | Updates existing todo when card changes |
| Fetch completed todos | HTTP Request | Retrieve completed todos for todolist | Check if New Card (false) | Combine all todos | Updates existing todo when card changes |
| Combine all todos | Merge | Combine active + completed todos | Fetch active todos; Fetch completed todos | Find todo and prepare update | Updates existing todo when card changes |
| Find todo and prepare update | Code | Find todo by embedded Fizzy ID; compute update flags | Combine all todos | Check if assignment change | Updates existing todo when card changes |
| Check if assignment change | IF | Route assignment updates vs status checks | Find todo and prepare update | Update todo assignees; Check if card closed | Handles assignment updates and completion status |
| Update todo assignees | Basecamp (community) | Update todo content/assignees/description | Check if assignment change (true) | — | Handles assignment updates and completion status |
| Check if card closed | IF | Route completion | Check if assignment change (false) | Mark todo as complete; Check if card reopened | Handles assignment updates and completion status |
| Mark todo as complete | Basecamp (community) | Complete todo | Check if card closed (true) | — | Handles assignment updates and completion status |
| Check if card reopened | IF | Route uncompletion for reopen/postpone/triage | Check if card closed (false) | Mark todo as incomplete | Handles assignment updates and completion status |
| Mark todo as incomplete | Basecamp (community) | Uncomplete todo | Check if card reopened (true) | — | Handles assignment updates and completion status |
| Setup instructions | Sticky Note | Documentation block | — | — | ## Setup steps  \n\n**1. Install Basecamp community node**\nInstall from: https://www.npmjs.com/package/n8n-nodes-basecamp  \n\n**2. Configure Basecamp credentials**\nCreate integration: https://launchpad.37signals.com/integrations\nAdd credentials in n8n with your Client ID and Client Secret  \n\n**3. Set your account ID**\nUpdate the \"Set Basecamp account ID\" node with your Account ID  \n\n**4. Prepare your Basecamp project**\n- Create project matching your Fizzy board name exactly\n- Enable Todos tool in project settings  \n\n**5. Configure Fizzy webhook**\n\n- Each board needs its own webhook:\n- Click globe icon in Fizzy board\n- Paste webhook URL from \"Receive Fizzy webhook\" node\n- Enable all events  \n\n## How it works\n\nThis workflow syncs Fizzy cards to Basecamp todos in real-time. When cards are created, assigned, or closed in Fizzy, the corresponding todos update automatically in Basecamp. Card tags determine the todolist, and assignees sync by email match. |
| Initial setup | Sticky Note | Documentation block | — | — | Receives webhook from Fizzy and fetches Basecamp data |
| Project matching | Sticky Note | Documentation block | — | — | Matches Fizzy Data to Basecamp Data (Projects, Todoset, Todolist) |
| Create path | Sticky Note | Documentation block | — | — | Creates new todo when fizzy card status is equal to published (New) |
| Update path | Sticky Note | Documentation block | — | — | Updates existing todo when card changes |
| Status changes | Sticky Note | Documentation block | — | — | Handles assignment updates and completion status |

---

## 4. Reproducing the Workflow from Scratch

1) **Install required community node**
1. In n8n, install community package: `n8n-nodes-basecamp`  
   Link: https://www.npmjs.com/package/n8n-nodes-basecamp

2) **Create Basecamp OAuth2 credentials**
1. Create a Basecamp integration: https://launchpad.37signals.com/integrations  
2. In n8n **Credentials**, create **Basecamp OAuth2 (basecamp4OAuth2Api)** with the Client ID/Secret from Basecamp.  
3. Complete OAuth authorization for the Basecamp account that contains your projects.

3) **Create the Webhook trigger**
1. Add **Webhook** node named **“Receive Fizzy webhook”**
   - Method: `POST`
   - Path: `fizzyxbasecamp`
   - Response Mode: `Last Node`
2. Copy the production webhook URL; you will paste it into Fizzy later.

4) **Add Basecamp account ID injection**
1. Add **Code** node named **“Set Basecamp account ID”** connected from the webhook.
2. In the code, set `BASECAMP_ACCOUNT_ID` to your numeric/actual Basecamp account ID.
3. Ensure the node outputs the original webhook JSON plus `basecampAccountId`.

5) **Fetch Basecamp projects and people (parallel)**
1. Add **HTTP Request** node **“Fetch Basecamp projects”**
   - Authentication: predefined credential → `basecamp4OAuth2Api`
   - URL: `https://3.basecampapi.com/{{ basecampAccountId }}/projects.json`  
     (use expression referencing “Set Basecamp account ID” output)
2. Add **HTTP Request** node **“Fetch Basecamp people”**
   - Authentication: same credential
   - URL: `https://3.basecampapi.com/{{ basecampAccountId }}/people.json`
3. Connect **“Set Basecamp account ID”** to both HTTP nodes.

6) **Merge projects + people**
1. Add **Merge** node **“Combine projects and people”**
2. Connect:
   - Fetch Basecamp projects → Merge input 0
   - Fetch Basecamp people → Merge input 1

7) **Match project and todoset**
1. Add **Code** node **“Match project and get todoset”**
2. Implement logic:
   - Read Fizzy board name from webhook payload (`body.board.name`)
   - Find Basecamp project with the same name (case-insensitive exact match)
   - Extract `projectId`
   - Find dock item with `name === 'todoset'` and extract `todosetId`
   - Pass along `fizzyAction`, original `webhookData`, and `basecampPeople` list
3. Connect Merge → this Code node.

8) **Fetch todolists**
1. Add **HTTP Request** node **“Fetch project todolists”**
   - Auth: Basecamp OAuth2 credential
   - URL: `https://3.basecampapi.com/{{ accountId }}/buckets/{{ projectId }}/todosets/{{ todosetId }}/todolists.json`
2. Connect “Match project and get todoset” → “Fetch project todolists”.

9) **Match todolist + assignees**
1. Add **Code** node **“Match todolist and assignees”**
2. Implement:
   - Determine `firstTag` = first Fizzy tag from `body.eventable.tags[0]`, else `"Fizzy Todo"`
   - Find todolist whose title matches `firstTag` (case-insensitive exact)
   - Map Fizzy assignees (`body.eventable.assignees[].email_address`) to Basecamp people emails (`people[].email_address`) to produce `assigneeIds`
   - Output base payload including `projectId`, `todosetId`, `todolistFound`, `todolistId` or `newTodolistName`, plus `fizzyCardId`, `cardTitle`, `cardDescription`, `fizzyAction`, `basecampAccountId`
3. Connect Fetch project todolists → this Code node.

10) **Check/create missing todolist**
1. Add **IF** node **“Check if todolist exists”**
   - Condition: `todolistFound` is true
2. False branch:
   - Add **Basecamp** node **“Create new todolist”**
     - Resource: `todolist`
     - Operation: `createTodolist`
     - Bucket ID = `projectId`
     - Todoset ID = `todosetId`
     - Name = `newTodolistName`
     - Description = “Auto-created from Fizzy”
   - Add **Code** node **“Add new todolist ID”** to merge created `id` into original payload as `todolistId`
3. Wiring:
   - Match todolist and assignees → IF
   - IF false → Create new todolist → Add new todolist ID
   - IF true → goes forward without creation

11) **Route create vs update**
1. Add **IF** node **“Check if New Card”**
   - Condition: `fizzyAction == card_published`
2. Connect:
   - IF true input from **Check if todolist exists (true)** and from **Add new todolist ID**
   - (Both paths feed the same IF node input)

12) **Create path (new card)**
1. Add **Basecamp** node **“Create new todo”**
   - Resource: `todo`
   - Operation: `createTodo`
   - Bucket ID = `projectId`
   - Todolist ID = `todolistId`
   - Content = `cardTitle`
   - Assignee IDs = `assigneeIds`
   - Notify = true
   - Description must include a stable marker line, e.g.:  
     `Fizzy ID: {fizzyCardId} - Please Don't Delete!` + blank line + `{cardDescription}`
2. Connect “Check if New Card” (true) → “Create new todo”.

13) **Update path (existing card) – fetch all todos**
1. Add **HTTP Request** node **“Fetch active todos”**
   - URL: `/buckets/{projectId}/todolists/{todolistId}/todos.json`
   - Auth: Basecamp credential
   - Configure pagination using Link header `rel="next"` extraction (as in JSON)
2. Add **HTTP Request** node **“Fetch completed todos”**
   - URL: same but with `?completed=true`
   - Same pagination
3. Add **Merge** node **“Combine all todos”**
4. Connect “Check if New Card” (false) → both HTTP nodes, then to Merge inputs 0/1.

14) **Find matching todo and compute update flags**
1. Add **Code** node **“Find todo and prepare update”**
   - Collect todos from `$input.all()`
   - Find the todo whose `description` contains `Fizzy ID: <fizzyCardId>` (regex)
   - Output `todoId` + `projectId` + `assigneeIds` + `cardTitle` + `cardDescription` + `fizzyAction`
   - Set status intent flags for close/reopen/postpone/triage actions
2. Connect Combine all todos → this Code node.

15) **Apply updates**
1. Add **IF** node **“Check if assignment change”**
   - OR: `fizzyAction == card_assigned` OR `card_unassigned`
2. True branch:
   - Add Basecamp node **“Update todo assignees”**
     - Operation: `updateTodo`
     - Update content/title, assignee IDs, and rewrite description preserving the “Fizzy ID” line
3. False branch:
   - Add **IF** node **“Check if card closed”** (`fizzyAction == card_closed`)
     - True → Basecamp node **“Mark todo as complete”** (completeTodo)
     - False → **IF** node **“Check if card reopened”** (OR: reopened/postponed/auto_postponed/sent_back_to_triage)
       - True → Basecamp node **“Mark todo as incomplete”** (uncompleteTodo)
4. Connect in order:
   - Find todo and prepare update → Check if assignment change
   - Assignment true → Update todo assignees
   - Assignment false → Check if card closed → (complete) or (reopened check) → (uncomplete)

16) **Configure Fizzy**
1. For each Fizzy board, create a webhook (Fizzy UI “globe” icon).
2. Paste the n8n webhook URL from “Receive Fizzy webhook”.
3. Enable all desired events (at minimum those referenced: published/assigned/unassigned/closed/reopened/postponed/auto_postponed/sent_back_to_triage).

**Required Basecamp-side constraints**
- Basecamp project name must match the Fizzy board name (case-insensitive exact match as implemented).
- Todos tool must be enabled in the project.
- Todolist titles should match Fizzy card tags (first tag), otherwise new lists are created.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Install Basecamp community node | https://www.npmjs.com/package/n8n-nodes-basecamp |
| Create Basecamp integration (Client ID/Secret) | https://launchpad.37signals.com/integrations |
| Operational note: mapping key depends on keeping “Fizzy ID: …” in the Basecamp todo description | If removed/edited, updates will fail because the workflow cannot find the matching todo |
| Naming convention requirement | Basecamp project name must match Fizzy board name for automatic mapping |
| Card tags drive todolist selection | First tag maps to todolist title; missing tag uses default “Fizzy Todo” and may create a list if not present |
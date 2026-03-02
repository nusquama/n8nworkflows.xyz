Update KlickTipp contact tags by tag names

https://n8nworkflows.xyz/workflows/update-klicktipp-contact-tags-by-tag-names-13663


# Update KlickTipp contact tags by tag names

This document provides a technical breakdown of the n8n workflow: **Update KlickTipp contact tags by tag names**.

---

### 1. Workflow Overview
This workflow functions as a reusable utility (sub-workflow) for KlickTipp tag management. Its primary purpose is to allow users to update contact tags using human-readable **tag names** rather than internal **tag IDs**. 

The workflow follows a 4-stage logic:
1.  **Input Reception:** Accepts an email address and two lists (Add/Remove) of tag names.
2.  **Tag Resolution:** Fetches the current master tag list from KlickTipp and maps the provided names to their respective IDs using custom logic.
3.  **Conditional Execution:** Branches the logic to handle additions and removals separately, ensuring API calls are only made if work is required.
4.  **Consolidation:** Aggregates the results of all API calls to return a final success status to the calling workflow.

---

### 2. Block-by-Block Analysis

#### 2.1 Input & Fetching
- **Overview:** Receives the data payload and retrieves the global tag dictionary from KlickTipp to enable name-to-ID mapping.
- **Nodes Involved:** `Input: Email + Tag names`, `Get tag list`.
- **Node Details:**
    - **Input: Email + Tag names (Execute Workflow Trigger):** Defines the interface. It expects `email` (string), `tagNamesToAdd` (array), and `tagNamesToRemove` (array).
    - **Get tag list (KlickTipp):** A "Get All" operation on the Tag resource. It retrieves all existing tags in the KlickTipp account.
    - **Failure handling:** If the KlickTipp API is unreachable or credentials fail here, the workflow terminates early.

#### 2.2 Tag Name Resolution
- **Overview:** A specialized logic block that matches the user-provided names against the IDs fetched in the previous step.
- **Nodes Involved:** `Compare desired vs current tags`.
- **Node Details:**
    - **Compare desired vs current tags (Code):** Runs JavaScript to normalize strings (trimming and lowercasing) for robust matching. It builds a lookup table where keys are tag names and values are IDs.
    - **Key Expressions:** It filters out any names that do not exist in the KlickTipp account to prevent API errors in subsequent steps.
    - **Output:** Returns two arrays: `tagIdsToAdd` and `tagIdsToRemove`.

#### 2.3 Tag Addition Logic
- **Overview:** Checks if there are tags to add and executes the update.
- **Nodes Involved:** `If tags to add`, `Add tags to contact`.
- **Node Details:**
    - **If tags to add (Filter):** Checks if the `tagIdsToAdd` array is not empty.
    - **Add tags to contact (KlickTipp):** Uses the "Tagging" resource. It passes the email and the array of IDs. KlickTipp handles multiple IDs in a single call for additions.
    - **Edge Case:** If a contact does not exist, KlickTipp behavior depends on account settings (usually creates the contact or returns an error).

#### 2.4 Tag Removal Logic
- **Overview:** Manages the removal of tags. Unlike addition, removals are processed individually to ensure precision.
- **Nodes Involved:** `If tags to remove`, `Split tags to remove`, `Remove tag from contact`, `Collect removal results`.
- **Node Details:**
    - **If tags to remove (Filter):** Validates if `tagIdsToRemove` contains data.
    - **Split tags to remove (Split Out):** Converts the array of IDs into individual items, as the "Untag" operation typically requires separate execution per tag.
    - **Remove tag from contact (KlickTipp):** Executes the "Untag" operation for each item produced by the split.
    - **Collect removal results (Aggregate):** Re-groups the individual API responses into a single list of success/failure booleans.

#### 2.5 Final Result Aggregation
- **Overview:** Merges the paths of the Add and Remove branches to provide a clean exit point.
- **Nodes Involved:** `Merge`, `Return result to parent workflow`.
- **Node Details:**
    - **Merge:** Synchronizes the data from both the "Add" branch and the "Remove" branch.
    - **Return result to parent workflow (Set):** Calculates a final `success` boolean. It uses logic to check if both the addition and all individual removals returned a successful status.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Input: Email + Tag names | Execute Workflow Trigger | Entry Point | None | Get tag list | ## Input |
| Get tag list | KlickTipp | Data Fetching | Input: Email + Tag names | Compare desired vs current tags | ## Fetch tag list |
| Compare desired vs current tags | Code | Data Mapping | Get tag list | If tags to add, If tags to remove | ## Compare tags and build actions |
| If tags to add | Filter | Logical Branch | Compare desired vs current tags | Add tags to contact | ## Add missing tags |
| Add tags to contact | KlickTipp | API Action | If tags to add | Merge | ## Add missing tags |
| If tags to remove | Filter | Logical Branch | Compare desired vs current tags | Split tags to remove | ## Remove extra tags |
| Split tags to remove | Split Out | Data Iterator | If tags to remove | Remove tag from contact | ## Remove extra tags |
| Remove tag from contact | KlickTipp | API Action | Split tags to remove | Collect removal results | ## Remove extra tags |
| Collect removal results | Aggregate | Data Consolidation | Remove tag from contact | Merge | ## Remove extra tags |
| Merge | Merge | Path Sync | Add tags to contact, Collect removal results | Return result to parent workflow | ## Compute final success (add + remove) |
| Return result to parent workflow | Set | Final Output | Merge | None | ## Compute final success (add + remove) |

---

### 4. Reproducing the Workflow from Scratch

1.  **Trigger Setup:** Create an "Execute Workflow" trigger. Define a JSON example containing `email`, `tagNamesToAdd` (array of strings), and `tagNamesToRemove` (array of strings).
2.  **Fetch Master Tags:** Add a KlickTipp node. Set the resource to "Tag" and operation to "Get All". Connect it to the trigger.
3.  **Implement Mapping Logic:** Add a Code node. Use JavaScript to:
    *   Iterate through the KlickTipp master tag list.
    *   Create a mapping object `{ "tag name": ID }`.
    *   Check the input arrays against this map to produce two new arrays of IDs.
4.  **Branching:** Add two Filter nodes connected to the Code node. 
    *   Filter A: `tagIdsToAdd` is not empty.
    *   Filter B: `tagIdsToRemove` is not empty.
5.  **Tag Addition:** Connect Filter A to a KlickTipp node (Resource: Contact Tagging, Operation: Tag). Map the email from the trigger and the ID array from the Code node.
6.  **Tag Removal:** 
    *   Connect Filter B to a "Split Out" node targeting the removal ID array.
    *   Connect the Split Out node to a KlickTipp node (Resource: Contact Tagging, Operation: Untag).
    *   Connect the KlickTipp node to an "Aggregate" node to collect the `success` field from all iterations.
7.  **Finalize:** Add a Merge node (Mode: Choose Branch/Combine) to bring the Add and Remove results together. Use a Set node to evaluate if all operations were successful and return this to the parent workflow.
8.  **Credentials:** Ensure all KlickTipp nodes use a valid KlickTipp API credential (User/Password).

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Introduction**: This workflow receives an email and tag name arrays, resolves them to IDs, and applies changes. | General introduction to workflow logic. |
| **Testing Requirement**: Test with a contact that already exists in KlickTipp. | Integration testing instruction. |
| **Non-existing Tags**: Non-existing tags in KlickTipp will be ignored unless "create tag" logic is added. | Functional constraint. |
| **Reusable Module**: Designed as a sub-workflow for consistent tagging behavior. | Architectural intent. |
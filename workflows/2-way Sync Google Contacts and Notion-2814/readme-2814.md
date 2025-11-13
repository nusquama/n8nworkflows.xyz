2-way Sync Google Contacts and Notion

https://n8nworkflows.xyz/workflows/2-way-sync-google-contacts-and-notion-2814


# 2-way Sync Google Contacts and Notion

### 1. Workflow Overview

This workflow provides a **two-way synchronization** between Google Contacts and a Notion database, ensuring that contact information is consistently mirrored and updated on both platforms. It supports filtering contacts by Google label or syncing all contacts, and it handles creation, updates, and deletions bidirectionally.

**Target Use Cases:**  
- Service providers managing client contact details  
- Individuals centralizing contacts in Notion  
- Automation enthusiasts integrating Google Contacts with Notion  

**Logical Blocks:**

- **1.1 Initial Import & Setup:** Import all or filtered Google Contacts into Notion initially.  
- **1.2 Notion Change Detection:** Triggered by Notion page creation, update, or deletion to sync changes to Google Contacts.  
- **1.3 Google Change Detection:** Periodic polling for Google Contacts updates to sync changes back to Notion.  
- **1.4 Contact Creation & Update Logic:** Handling creation of new contacts and updating existing ones on both platforms.  
- **1.5 Deletion Handling:** Detecting deletions on either side and deleting the corresponding contact/page on the other.  
- **1.6 Utility & State Management:** Managing sync tokens, ETags, and global variables to track sync state and avoid conflicts.

---

### 2. Block-by-Block Analysis

#### 1.1 Initial Import & Setup

- **Overview:**  
  This block imports Google Contacts (optionally filtered by group/label) and creates corresponding entries in Notion, initializing the sync.

- **Nodes Involved:**  
  - Globals  
  - Get all contacts  
  - Filter by Group  
  - Extract phones and addresses1  
  - Save contact1  
  - Set lastUpdatedByAutomation1  
  - Save ETag2  
  - List all groups1  
  - Split Groups  
  - Organize fields  
  - Add Google ID  
  - Google | Add contact to specific group  
  - Save ETag1  

- **Node Details:**  
  - **Globals:** Sets initial global variables or parameters for the import process.  
  - **Get all contacts:** Retrieves all Google Contacts.  
  - **Filter by Group:** Optionally filters contacts by a specific Google label/group.  
  - **Extract phones and addresses1:** Processes raw contact data to extract phone numbers and addresses into structured fields.  
  - **Save contact1:** Creates or updates the corresponding Notion page for each contact.  
  - **Set lastUpdatedByAutomation1:** Sends an HTTP request to Google API to mark the contact as last updated by automation (to avoid loops).  
  - **Save ETag2:** Saves the Notion page ETag for concurrency control.  
  - **List all groups1:** Retrieves all Google Contact groups for filtering or assignment.  
  - **Split Groups:** Splits the groups array to process individually.  
  - **Organize fields:** Prepares fields for Google API calls.  
  - **Add Google ID:** Adds the Google Contact ID to the Notion page for reference.  
  - **Google | Add contact to specific group:** Adds the new contact to a specific Google group.  
  - **Save ETag1:** Saves the ETag after adding the contact to the group.

- **Key Expressions/Variables:**  
  - Filtering by label/group name or ID.  
  - Extracted phone and address fields mapped to Notion properties.  
  - Use of ETags and lastUpdatedByAutomation flags to prevent sync loops.

- **Input/Output Connections:**  
  - Starts from Globals → Get all contacts → Filter by Group → Extract phones and addresses1 → Save contact1 → Set lastUpdatedByAutomation1 → Save ETag2.  
  - Parallel branch for group handling: List all groups1 → Split Groups → Organize fields → Add Google ID → Google | Add contact to specific group → Save ETag1.

- **Potential Failures:**  
  - Google API rate limits or auth errors.  
  - Notion API concurrency conflicts (ETag mismatch).  
  - Missing or invalid group labels causing filter to fail.  
  - Data extraction errors if contact fields are missing or malformed.

---

#### 1.2 Notion Change Detection

- **Overview:**  
  Listens for Notion page creations, updates, and deletions to propagate changes to Google Contacts.

- **Nodes Involved:**  
  - Page Created (Notion Trigger)  
  - Page updated (Notion Trigger)  
  - Page deleted (Notion Trigger)  
  - Merge1  
  - Page was edited by user (Filter)  
  - new contact (If)  
  - Find contact (Google Contacts)  
  - Google | Create Contact (HTTP Request)  
  - Google | Update Contact (HTTP Request)  
  - Save ETag  
  - Save ETag1  
  - Delete contact (Google Contacts)  
  - Globals2  
  - Merge1  

- **Node Details:**  
  - **Page Created / Page updated:** Triggered when a Notion page representing a contact is created or updated.  
  - **Page deleted:** Triggered when a Notion page is deleted.  
  - **Page was edited by user:** Filters out changes not made by automation to avoid loops.  
  - **new contact:** Determines if the contact is new or existing.  
  - **Find contact:** Searches for the corresponding Google Contact by ID.  
  - **Google | Create Contact:** Creates a new Google Contact if none exists.  
  - **Google | Update Contact:** Updates an existing Google Contact.  
  - **Save ETag / Save ETag1:** Updates Notion page ETags after changes.  
  - **Delete contact:** Deletes the Google Contact if the Notion page was deleted.  
  - **Globals2:** Sets variables for this sync path.  
  - **Merge1:** Merges created and updated page flows for unified processing.

- **Key Expressions/Variables:**  
  - Checks if the Notion page was edited by a user or automation.  
  - Uses Google Contact ID stored in Notion to find/update contacts.  
  - Uses ETags to manage concurrency.

- **Input/Output Connections:**  
  - Page Created and Page updated → Merge1 → Page was edited by user → new contact → Google | Create Contact or Find contact → Google | Update Contact → Save ETag.  
  - Page deleted → Delete contact.

- **Potential Failures:**  
  - Notion trigger misfires or missed events.  
  - Google API auth or rate limits.  
  - Failure to find Google Contact by ID.  
  - Deletion nodes disabled may cause desync.  
  - Concurrency conflicts with ETags.

---

#### 1.3 Google Change Detection

- **Overview:**  
  Periodically polls Google Contacts for updates using sync tokens, then updates Notion accordingly.

- **Nodes Involved:**  
  - Every 1min1 (Schedule Trigger)  
  - Globals4  
  - Retrieve Sync Token (Notion)  
  - Google | Get updates (HTTP Request)  
  - Get syncToken (SplitOut)  
  - Split Contacts (SplitOut)  
  - Filter by Group2  
  - Only type CONTACT1  
  - Find contacts1 (Notion)  
  - Merge2  
  - Exists in Notion1 (If)  
  - Contact still exists (If)  
  - Contact was edited in Google1 (Filter)  
  - Extract phones and addresses4  
  - Has data (Filter)  
  - Update contact (Notion)  
  - Google | Set lastUpdatedByAutomation3 (HTTP Request)  
  - Update ETag3 (Notion)  
  - Delete contact2 (Notion)  
  - Only existing contacts (Filter)  
  - Extract phones and addresses3  
  - Save contact2  
  - Google | Set lastUpdatedByAutomation (HTTP Request)  
  - Update ETag2 (Notion)  

- **Node Details:**  
  - **Every 1min1:** Triggers the sync every minute.  
  - **Globals4:** Sets variables for this sync path.  
  - **Retrieve Sync Token:** Gets the last Google sync token stored in Notion.  
  - **Google | Get updates:** Calls Google API with sync token to get incremental updates.  
  - **Get syncToken:** Extracts new sync token from response.  
  - **Split Contacts:** Splits updated contacts for processing.  
  - **Filter by Group2:** Filters contacts by group/label if configured.  
  - **Only type CONTACT1:** Filters only contacts (not other Google People API types).  
  - **Find contacts1:** Searches for corresponding Notion pages.  
  - **Merge2:** Merges found and not found contacts.  
  - **Exists in Notion1:** Checks if contact exists in Notion.  
  - **Contact still exists:** Checks if contact still exists in Google.  
  - **Contact was edited in Google1:** Filters contacts edited by user (not automation).  
  - **Extract phones and addresses4:** Extracts fields for Notion update.  
  - **Has data:** Filters contacts with data to update.  
  - **Update contact:** Updates Notion page with Google data.  
  - **Google | Set lastUpdatedByAutomation3:** Marks Google contact as updated by automation.  
  - **Update ETag3:** Updates Notion ETag.  
  - **Delete contact2:** Deletes Notion page if Google contact deleted.  
  - **Only existing contacts:** Filters contacts that exist in Notion.  
  - **Extract phones and addresses3:** Extracts fields for Notion save.  
  - **Save contact2:** Saves new or updated contact in Notion.  
  - **Google | Set lastUpdatedByAutomation:** Marks Google contact as updated by automation.  
  - **Update ETag2:** Updates Notion ETag.

- **Key Expressions/Variables:**  
  - Use of Google sync tokens to fetch incremental updates.  
  - Filtering by group and contact type.  
  - Use of ETags and lastUpdatedByAutomation flags.  
  - Conditional deletion if contact no longer exists.

- **Input/Output Connections:**  
  - Every 1min1 → Globals4 → Retrieve Sync Token → Google | Get updates → Get syncToken + Split Contacts → Filter by Group2 → Only type CONTACT1 → Find contacts1 → Merge2 → Exists in Notion1 → Contact still exists → Contact was edited in Google1 → Extract phones and addresses4 → Has data → Update contact → Google | Set lastUpdatedByAutomation3 → Update ETag3.  
  - Contact still exists (false) → Delete contact2.  
  - Only existing contacts → Extract phones and addresses3 → Save contact2 → Google | Set lastUpdatedByAutomation → Update ETag2.

- **Potential Failures:**  
  - Google API rate limits or auth errors.  
  - Sync token expiration or invalidation.  
  - Notion API concurrency conflicts.  
  - Data extraction errors.  
  - Missed deletions if deletion nodes disabled.

---

#### 1.4 Contact Creation & Update Logic

- **Overview:**  
  Handles the logic for creating new contacts or updating existing contacts on both Google and Notion sides.

- **Nodes Involved:**  
  - new contact (If)  
  - Google | Create Contact (HTTP Request)  
  - Add Google ID (Notion)  
  - Google | Update Contact (HTTP Request)  
  - Save ETag  
  - Save ETag1  
  - Set lastUpdatedByAutomation1  
  - Set lastUpdatedByAutomation3  
  - Update contact (Notion)  
  - Has data (Filter)  
  - Extract phones and addresses1,3,4  
  - Save contact1,2  

- **Node Details:**  
  - **new contact:** Determines if the contact is new or existing based on presence of Google ID.  
  - **Google | Create Contact:** Creates a new contact in Google Contacts.  
  - **Add Google ID:** Adds the Google Contact ID to the Notion page after creation.  
  - **Google | Update Contact:** Updates an existing Google Contact.  
  - **Save ETag / Save ETag1:** Saves Notion page ETags after updates.  
  - **Set lastUpdatedByAutomation1 / 3:** Marks Google contacts as updated by automation.  
  - **Update contact:** Updates Notion page with new data.  
  - **Has data:** Filters contacts with data to update.  
  - **Extract phones and addresses:** Extracts and formats phone and address fields for syncing.  
  - **Save contact:** Saves contact data to Notion.

- **Key Expressions/Variables:**  
  - Checks for existence of Google ID to branch creation vs update.  
  - Uses lastUpdatedByAutomation flags to prevent sync loops.  
  - Extracts and maps phone/address fields carefully.

- **Input/Output Connections:**  
  - Branches from Notion triggers or Google updates.  
  - Creates or updates contacts on Google and Notion accordingly.

- **Potential Failures:**  
  - API errors during creation or update.  
  - Missing or malformed data causing failures.  
  - Sync loops if lastUpdatedByAutomation flags not set properly.

---

#### 1.5 Deletion Handling

- **Overview:**  
  Detects deletions on either Google or Notion side and deletes the corresponding contact/page on the other side.

- **Nodes Involved:**  
  - Page deleted (Notion Trigger)  
  - Delete contact (Google Contacts)  
  - Deleted pages (Filter)  
  - Delete contact1 (Google Contacts)  
  - Contact still exists (If)  
  - Delete contact2 (Notion)  

- **Node Details:**  
  - **Page deleted:** Triggered when a Notion page is deleted.  
  - **Delete contact:** Deletes the corresponding Google Contact.  
  - **Deleted pages:** Filters deleted Notion pages.  
  - **Delete contact1:** Deletes Google Contact for deleted Notion pages.  
  - **Contact still exists:** Checks if Google contact still exists.  
  - **Delete contact2:** Deletes Notion page if Google contact was deleted.

- **Key Expressions/Variables:**  
  - Uses Google Contact ID stored in Notion to identify contacts to delete.  
  - Conditional execution depending on whether deletion nodes are enabled.

- **Input/Output Connections:**  
  - Notion page deletion → Delete contact (Google).  
  - Google contact deletion detected → Delete Notion page.

- **Potential Failures:**  
  - Deletion nodes disabled causing desync.  
  - API errors deleting contacts or pages.  
  - Race conditions if deletions happen simultaneously.

---

#### 1.6 Utility & State Management

- **Overview:**  
  Manages global variables, sync tokens, ETags, and other stateful data to ensure consistent synchronization and avoid conflicts.

- **Nodes Involved:**  
  - Globals, Globals2, Globals3, Globals4 (Set nodes)  
  - Retrieve Sync Token (Notion)  
  - Update Sync Token (Notion)  
  - Save ETag, Save ETag1, Save ETag2, Save ETag3 (Notion)  
  - Get syncToken (SplitOut)  
  - Split userDefined, Split Contacts, Split Groups (SplitOut)  

- **Node Details:**  
  - **Globals nodes:** Initialize or update workflow-wide variables for different sync paths.  
  - **Retrieve Sync Token:** Reads the last Google sync token from Notion to fetch incremental updates.  
  - **Update Sync Token:** Saves the new sync token after fetching updates.  
  - **Save ETag nodes:** Save Notion page ETags to manage concurrency and avoid overwriting changes.  
  - **SplitOut nodes:** Split arrays of contacts, groups, or tokens for individual processing.

- **Key Expressions/Variables:**  
  - Sync tokens for incremental Google API calls.  
  - ETags for Notion concurrency control.  
  - Global variables for configuration and flow control.

- **Input/Output Connections:**  
  - Used throughout the workflow to maintain state and enable incremental sync.

- **Potential Failures:**  
  - Lost or corrupted sync tokens causing full resyncs.  
  - ETag mismatches causing update conflicts.  
  - Missing global variables causing logic errors.

---

### 3. Summary Table

| Node Name                      | Node Type             | Functional Role                          | Input Node(s)                     | Output Node(s)                          | Sticky Note                                                                                          |
|--------------------------------|-----------------------|----------------------------------------|----------------------------------|---------------------------------------|----------------------------------------------------------------------------------------------------|
| Globals                        | Set                   | Initialize variables for initial import| -                                | Get all contacts                      |                                                                                                    |
| Get all contacts               | Google Contacts        | Retrieve all Google Contacts            | Globals                         | Filter by Group                       |                                                                                                    |
| Filter by Group                | Filter                | Filter contacts by Google label/group  | Get all contacts                | Extract phones and addresses1         |                                                                                                    |
| Extract phones and addresses1  | Set                   | Extract phone and address fields        | Filter by Group                 | Save contact1                        |                                                                                                    |
| Save contact1                 | Notion                | Create/update Notion page for contact   | Extract phones and addresses1   | Set lastUpdatedByAutomation1          |                                                                                                    |
| Set lastUpdatedByAutomation1  | HTTP Request          | Mark Google contact as updated by automation | Save contact1                | Save ETag2                          |                                                                                                    |
| Save ETag2                   | Notion                | Save Notion page ETag                    | Set lastUpdatedByAutomation1    | -                                    |                                                                                                    |
| List all groups1              | HTTP Request          | Retrieve Google Contact groups           | -                              | Split Groups                        |                                                                                                    |
| Split Groups                 | SplitOut              | Split groups array for processing        | List all groups1                | Organize fields                    |                                                                                                    |
| Organize fields              | Set                   | Prepare fields for Google API             | Split Groups                   | Add Google ID                     |                                                                                                    |
| Add Google ID                | Notion                | Add Google Contact ID to Notion page      | Google | Create Contact         | Google | Add contact to specific group |                                                                                                    |
| Google | Add contact to specific group | HTTP Request          | Add contact to Google group               | Add Google ID                  | Save ETag1                       |                                                                                                    |
| Save ETag1                   | Notion                | Save Notion page ETag after group add     | Google | Add contact to specific group | -                                |                                                                                                    |
| Page Created                 | Notion Trigger        | Trigger on Notion page creation           | -                              | Merge1                            |                                                                                                    |
| Page updated                 | Notion Trigger        | Trigger on Notion page update             | -                              | Merge1                            |                                                                                                    |
| Merge1                      | Merge                 | Merge created and updated page flows      | Page Created, Page updated     | Page was edited by user           |                                                                                                    |
| Page was edited by user      | Filter                | Filter out automation edits                | Merge1                        | new contact                      |                                                                                                    |
| new contact                 | If                    | Check if contact is new or existing        | Page was edited by user        | Google | Create Contact, Page deleted |                                                                                                    |
| Google | Create Contact      | HTTP Request          | Create new Google Contact                  | new contact                   | Add Google ID                    |                                                                                                    |
| Find contact                | Google Contacts        | Find existing Google Contact by ID         | Page deleted, new contact      | Google | Update Contact           |                                                                                                    |
| Google | Update Contact      | HTTP Request          | Update existing Google Contact              | Find contact                  | Save ETag                       |                                                                                                    |
| Save ETag                   | Notion                | Save Notion page ETag after update          | Google | Update Contact          | -                                |                                                                                                    |
| Page deleted                | If                    | Detect Notion page deletion                  | new contact                   | Delete contact, Find contact      |                                                                                                    |
| Delete contact              | Google Contacts        | Delete Google Contact                         | Page deleted                  | -                                |                                                                                                    |
| Globals2                    | Set                   | Set variables for Notion change sync        | Every 1min1                   | Page was edited by user           |                                                                                                    |
| Every 1min1                 | Schedule Trigger       | Trigger sync every minute                    | -                            | Globals4                        |                                                                                                    |
| Globals4                    | Set                   | Set variables for Google change sync        | Every 1min1                   | Retrieve Sync Token              |                                                                                                    |
| Retrieve Sync Token         | Notion                | Get last Google sync token                    | Globals4                     | Google | Get updates             |                                                                                                    |
| Google | Get updates         | HTTP Request          | Get incremental Google Contacts updates      | Retrieve Sync Token           | Get syncToken, Split Contacts    |                                                                                                    |
| Get syncToken               | SplitOut              | Extract new sync token                        | Google | Get updates             | Update Sync Token               |                                                                                                    |
| Update Sync Token           | Notion                | Save new Google sync token                     | Get syncToken                | -                              |                                                                                                    |
| Split Contacts             | SplitOut              | Split updated contacts array                   | Google | Get updates             | Filter by Group2               |                                                                                                    |
| Filter by Group2            | Filter                | Filter contacts by group/label                  | Split Contacts              | Only type CONTACT1             |                                                                                                    |
| Only type CONTACT1          | Filter                | Filter only contacts (exclude other types)      | Filter by Group2             | Find contacts1, Merge2         |                                                                                                    |
| Find contacts1             | Notion                | Find corresponding Notion pages for contacts    | Only type CONTACT1           | Merge2                        |                                                                                                    |
| Merge2                     | Merge                 | Merge found and not found contacts               | Find contacts1, Only type CONTACT1 | Exists in Notion1           |                                                                                                    |
| Exists in Notion1          | If                    | Check if contact exists in Notion                | Merge2                      | Contact still exists, Only existing contacts |                                                                                                    |
| Contact still exists       | If                    | Check if Google contact still exists              | Exists in Notion1            | Contact was edited in Google1, Delete contact2 |                                                                                                    |
| Contact was edited in Google1 | Filter                | Filter contacts edited by user (not automation)   | Contact still exists         | Extract phones and addresses4  |                                                                                                    |
| Extract phones and addresses4 | Set                   | Extract fields for Notion update                   | Contact was edited in Google1 | Has data                      |                                                                                                    |
| Has data                   | Filter                | Filter contacts with data to update                 | Extract phones and addresses4 | Update contact                |                                                                                                    |
| Update contact             | Notion                | Update Notion page with Google data                 | Has data                    | Google | Set lastUpdatedByAutomation3 |                                                                                                    |
| Google | Set lastUpdatedByAutomation3 | HTTP Request          | Mark Google contact as updated by automation        | Update contact              | Update ETag3                  |                                                                                                    |
| Update ETag3               | Notion                | Save Notion page ETag after update                   | Google | Set lastUpdatedByAutomation3 | -                            |                                                                                                    |
| Delete contact2            | Notion                | Delete Notion page if Google contact deleted         | Contact still exists (false) | -                            |                                                                                                    |
| Only existing contacts     | Filter                | Filter contacts existing in Notion                     | Exists in Notion1            | Extract phones and addresses3 |                                                                                                    |
| Extract phones and addresses3 | Set                   | Extract fields for Notion save                        | Only existing contacts       | Save contact2                |                                                                                                    |
| Save contact2              | Notion                | Save new or updated contact in Notion                  | Extract phones and addresses3 | Google | Set lastUpdatedByAutomation |                                                                                                    |
| Google | Set lastUpdatedByAutomation | HTTP Request          | Mark Google contact as updated by automation            | Save contact2               | Update ETag2                  |                                                                                                    |
| Update ETag2               | Notion                | Save Notion page ETag after update                       | Google | Set lastUpdatedByAutomation | -                            |                                                                                                    |
| When clicking ‘Test workflow’ | Manual Trigger         | Manual trigger for testing workflow                      | -                            | Globals3                      |                                                                                                    |
| Globals3                   | Set                   | Set variables for manual test trigger                    | When clicking ‘Test workflow’ | Get all contacts2             |                                                                                                    |
| Get all contacts2          | Google Contacts        | Retrieve all Google Contacts for manual test              | Globals3                     | Split userDefined            |                                                                                                    |
| Split userDefined          | SplitOut              | Split contacts array for manual test processing             | Get all contacts2             | Only contacts that were automated |                                                                                                    |
| Only contacts that were automated | Filter                | Filter contacts marked as updated by automation             | Split userDefined            | Find contacts in Notion      |                                                                                                    |
| Find contacts in Notion    | Notion                | Find corresponding Notion pages for filtered contacts       | Only contacts that were automated | Deleted pages              |                                                                                                    |
| Deleted pages             | Filter                | Filter deleted Notion pages                                  | Find contacts in Notion      | Delete contact1             |                                                                                                    |
| Delete contact1           | Google Contacts        | Delete Google contacts for deleted Notion pages               | Deleted pages                | -                            |                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Credentials:**  
   - Set up Google Contacts OAuth2 credentials.  
   - Set up Notion API credentials with access to the target database.

2. **Initial Import Block:**  
   - Create a **Set** node named `Globals` to define initial parameters (e.g., group filters).  
   - Add a **Google Contacts** node `Get all contacts` to fetch all contacts.  
   - Add a **Filter** node `Filter by Group` to optionally filter contacts by label.  
   - Add a **Set** node `Extract phones and addresses1` to parse phone numbers and addresses into structured fields.  
   - Add a **Notion** node `Save contact1` to create or update Notion pages with contact data.  
   - Add an **HTTP Request** node `Set lastUpdatedByAutomation1` to mark Google contacts as updated by automation.  
   - Add a **Notion** node `Save ETag2` to save the Notion page ETag.  
   - Add an **HTTP Request** node `List all groups1` to fetch Google contact groups.  
   - Add a **SplitOut** node `Split Groups` to process groups individually.  
   - Add a **Set** node `Organize fields` to prepare group assignment fields.  
   - Add a **Notion** node `Add Google ID` to store Google Contact ID in Notion.  
   - Add an **HTTP Request** node `Google | Add contact to specific group` to assign contacts to groups.  
   - Add a **Notion** node `Save ETag1` to save ETag after group assignment.

3. **Notion Change Detection Block:**  
   - Add **Notion Trigger** nodes `Page Created`, `Page updated`, and `Page deleted` to listen for changes.  
   - Add a **Merge** node `Merge1` to combine created and updated flows.  
   - Add a **Filter** node `Page was edited by user` to exclude automation edits.  
   - Add an **If** node `new contact` to check if Google ID exists.  
   - Add a **Google Contacts** node `Find contact` to find existing Google contacts by ID.  
   - Add **HTTP Request** nodes `Google | Create Contact` and `Google | Update Contact` for creation and update.  
   - Add **Notion** nodes `Save ETag` and `Save ETag1` to save ETags after updates.  
   - Add a **Google Contacts** node `Delete contact` to delete Google contacts when Notion pages are deleted.

4. **Google Change Detection Block:**  
   - Add a **Schedule Trigger** node `Every 1min1` to poll Google every minute.  
   - Add a **Set** node `Globals4` for variables.  
   - Add a **Notion** node `Retrieve Sync Token` to get the last sync token.  
   - Add an **HTTP Request** node `Google | Get updates` to fetch incremental updates using the sync token.  
   - Add a **SplitOut** node `Get syncToken` to extract the new sync token.  
   - Add a **Notion** node `Update Sync Token` to save the new sync token.  
   - Add a **SplitOut** node `Split Contacts` to process each updated contact.  
   - Add a **Filter** node `Filter by Group2` to filter contacts by group.  
   - Add a **Filter** node `Only type CONTACT1` to filter only contacts.  
   - Add a **Notion** node `Find contacts1` to find corresponding Notion pages.  
   - Add a **Merge** node `Merge2` to merge found and not found contacts.  
   - Add an **If** node `Exists in Notion1` to check if contact exists in Notion.  
   - Add an **If** node `Contact still exists` to check if Google contact exists.  
   - Add a **Filter** node `Contact was edited in Google1` to filter user edits.  
   - Add a **Set** node `Extract phones and addresses4` to extract fields for Notion update.  
   - Add a **Filter** node `Has data` to filter contacts with data.  
   - Add a **Notion** node `Update contact` to update Notion pages.  
   - Add an **HTTP Request** node `Google | Set lastUpdatedByAutomation3` to mark Google contacts as updated.  
   - Add a **Notion** node `Update ETag3` to save ETag.  
   - Add a **Notion** node `Delete contact2` to delete Notion pages if Google contact deleted.  
   - Add a **Filter** node `Only existing contacts` to filter contacts existing in Notion.  
   - Add a **Set** node `Extract phones and addresses3` to extract fields for Notion save.  
   - Add a **Notion** node `Save contact2` to save contacts in Notion.  
   - Add an **HTTP Request** node `Google | Set lastUpdatedByAutomation` to mark Google contacts as updated.  
   - Add a **Notion** node `Update ETag2` to save ETag.

5. **Deletion Handling Block:**  
   - Connect `Page deleted` trigger to `Delete contact` Google Contacts node.  
   - Add a **Filter** node `Deleted pages` to detect deleted Notion pages.  
   - Add a **Google Contacts** node `Delete contact1` to delete Google contacts for deleted Notion pages.  
   - Add an **If** node `Contact still exists` to check Google contact existence.  
   - Add a **Notion** node `Delete contact2` to delete Notion pages for deleted Google contacts.

6. **Utility & State Management:**  
   - Add **Set** nodes `Globals`, `Globals2`, `Globals3`, `Globals4` to manage variables for different sync paths.  
   - Use **SplitOut** nodes to split arrays for processing.  
   - Use **Notion** nodes to retrieve and update sync tokens and ETags.

7. **Manual Testing:**  
   - Add a **Manual Trigger** node `When clicking ‘Test workflow’` connected to `Globals3` and subsequent nodes for manual testing.

**Credentials:**  
- Google Contacts OAuth2 with appropriate scopes (contacts.readonly, contacts).  
- Notion API integration with read/write access to the contacts database.

**Default Values & Constraints:**  
- Sync interval set to 1 minute by default.  
- Optional filtering by Google contact groups/labels.  
- Deletion nodes can be disabled to prevent automatic deletions.  
- ETag and sync token management to avoid conflicts and enable incremental sync.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                      |
|----------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| Sync is bidirectional: deleting a contact on one side deletes it on the other by default.          | Workflow description                                |
| Deletion behavior can be disabled by disabling deletion nodes to prevent accidental data loss.     | Workflow description                                |
| Initial import is required to bring Google Contacts into Notion before continuous sync starts.     | Workflow description                                |
| Creator’s other templates available at: [https://n8n.io/creators/solomon/](https://n8n.io/creators/solomon/) | External link                                       |
| Workflow includes detailed instructions embedded within nodes for setup and customization.         | Workflow description                                |
| Screenshots included in original workflow for UI reference (not included here).                     | Provided images in original description             |

---

This document fully describes the workflow’s structure, logic, and node configurations to enable understanding, reproduction, and modification by advanced users or AI agents.
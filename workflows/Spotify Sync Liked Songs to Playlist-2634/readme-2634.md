Spotify Sync Liked Songs to Playlist

https://n8nworkflows.xyz/workflows/spotify-sync-liked-songs-to-playlist-2634


# Spotify Sync Liked Songs to Playlist

### 1. Workflow Overview

This workflow automates the synchronization of a Spotify user's Liked Songs with a custom Spotify playlist, ensuring that the playlist always mirrors the current state of the Liked Songs. It adds newly liked tracks and removes songs that have been unliked, maintaining a shareable playlist that reflects the user’s favorites.

**Target Use Case:**  
Spotify users who want a dedicated playlist that automatically reflects their Liked Songs, facilitating sharing or further playlist management.

**Logical Blocks:**

- **1.1 Input Reception & Initialization**  
  Trigger nodes to start the workflow manually or on a schedule; initial variable settings.

- **1.2 Data Retrieval**  
  Fetch Liked Songs, all user playlists, and tracks from the target playlist.

- **1.3 Playlist Identification**  
  Filter the user’s playlists to find the target playlist by name and extract its URI.

- **1.4 Data Preparation & Sorting**  
  Sort Liked Songs and playlist tracks by addition date for consistent comparison.

- **1.5 Data Comparison**  
  Compare Liked Songs with the playlist tracks to identify missing and extra songs.

- **1.6 Sync - Add Missing Songs**  
  Loop through missing songs and add them to the playlist.

- **1.7 Sync - Remove Unliked Songs**  
  Loop through songs in the playlist that are no longer liked and remove them.

- **1.8 Summary & Notifications**  
  Count added and deleted songs, send optional Gotify notifications about sync results.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Initialization

- **Overview:**  
  Initiates workflow execution either manually or on a fixed schedule. Sets essential variables including the target playlist name and records the workflow start time.

- **Nodes Involved:**  
  - Start (Manual Trigger)  
  - Schedule Trigger  
  - Edit set Vars  
  - Edit set intern vars

- **Node Details:**

| Node Name        | Details                                                                                                                                                                                                                   |
|------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Start            | Type: Manual Trigger. Role: Allows manual execution of the workflow. No parameters. Output connects to "Edit set Vars".                                                                                                     |
| Schedule Trigger | Type: Schedule Trigger (interval every 24 hours). Role: Automates workflow execution daily at 00:00. Outputs to "Edit set Vars".                                                                                        |
| Edit set Vars    | Type: Set Node. Role: Defines variables for workflow, notably `varplaylistto` which holds the target playlist name to sync with. The default value is `"CHANGE MEEEEEEEEE"` and must be replaced by the user with their playlist name. |
| Edit set intern vars | Type: Set Node. Role: Sets internal variables such as `timestart` capturing the Unix timestamp at workflow start for duration calculation. Outputs to "Spotify get Liked Songs" and "Spotify get all playlists".           |

- **Edge Cases / Failures:**  
  - Missing or incorrect playlist name in `varplaylistto` causes filtering failure downstream.  
  - Timezone misconfiguration could affect scheduled trigger timing.

---

#### 1.2 Data Retrieval

- **Overview:**  
  Retrieves the user's Liked Songs, all user playlists, and tracks from the target playlist.

- **Nodes Involved:**  
  - Spotify get Liked Songs  
  - Spotify get all playlists  
  - Spotify get Tracks of X

- **Node Details:**

| Node Name            | Details                                                                                                                                                                                                                              |
|----------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Spotify get Liked Songs | Type: Spotify API node. Role: Retrieves all liked songs from the user's Spotify library (`resource: library`). Configured to return all tracks without pagination issues (`returnAll: true`). Output to "Sort first added to first item". |
| Spotify get all playlists | Type: Spotify API node. Role: Fetches all user playlists (`operation: getUserPlaylists`). Returns all playlists for filtering. Output to "Filter Playlist x".                                                                        |
| Spotify get Tracks of X | Type: Spotify API node. Role: Fetches all tracks from the identified playlist (ID from `Set pluri`). Returns all tracks for comparison. Output to "Sort".                                                                             |

- **Edge Cases / Failures:**  
  - API authentication errors if Spotify credentials are not set or expired.  
  - Large libraries and playlists may cause timeouts or rate limiting.  
  - Playlists without tracks or empty liked songs handled gracefully but result in empty downstream data.

---

#### 1.3 Playlist Identification

- **Overview:**  
  Filters the fetched playlists to find the one matching the user-defined name and extracts its URI for further use.

- **Nodes Involved:**  
  - Filter Playlist x  
  - Set pluri

- **Node Details:**

| Node Name         | Details                                                                                                                                                          |
|-------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Filter Playlist x | Type: Filter Node. Role: Filters playlists to those whose name exactly matches `varplaylistto` variable. Essential to identify the target playlist. Output to "Set pluri". |
| Set pluri         | Type: Set Node. Role: Sets the playlist URI (`setpluri`) from the filtered playlist's `uri` property, used as ID in subsequent calls. Output to "Spotify get Tracks of X" and "Merge". |

- **Edge Cases / Failures:**  
  - Playlist name mismatch leads to empty output and failure in subsequent steps.  
  - Case sensitivity is strict; users must enter exact playlist names.  
  - Multiple playlists with the same name: first match is used, which may cause unexpected behavior.

---

#### 1.4 Data Preparation & Sorting

- **Overview:**  
  Sorts Liked Songs and playlist tracks by their addition date (`added_at`) to prepare for accurate comparison.

- **Nodes Involved:**  
  - Sort first added to first item  
  - Sort  
  - Merge

- **Node Details:**

| Node Name              | Details                                                                                                                                                              |
|------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Sort first added to first item | Type: Sort Node. Role: Sorts Liked Songs ascending by `added_at` date to maintain chronological order. Output to "Merge".                                      |
| Sort                   | Type: Sort Node. Role: Sorts playlist tracks ascending by `added_at`. Output to "Compare Datasets".                                                                |
| Merge                  | Type: Merge Node. Mode: Combine, multiplexes Liked Songs and playlist tracks arrays to prepare data for comparison. Output to "Compare Datasets".                    |

- **Edge Cases / Failures:**  
  - Missing or malformed `added_at` timestamps could cause sorting errors.  
  - Empty input arrays handled without error but lead to empty comparisons.

---

#### 1.5 Data Comparison

- **Overview:**  
  Compares the Liked Songs dataset with the playlist tracks dataset to identify songs that need to be added or deleted.

- **Nodes Involved:**  
  - Compare Datasets  
  - END (NoOp)

- **Node Details:**

| Node Name       | Details                                                                                                                                                               |
|-----------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Compare Datasets | Type: Compare Datasets Node. Role: Matches Liked Songs and playlist tracks by `track.uri` field. Outputs three branches: missing tracks (to add), extra tracks (to delete), and matched tracks. Outputs to loops and END. |
| END             | Type: NoOp Node. Role: Marks end of certain branches.                                                                                                                |

- **Edge Cases / Failures:**  
  - Comparison depends on correct track URI presence; missing URIs cause misclassification.  
  - Large datasets may impact performance.

---

#### 1.6 Sync - Add Missing Songs

- **Overview:**  
  Iterates over songs present in Liked Songs but missing in the playlist and adds them.

- **Nodes Involved:**  
  - Loop add missing  
  - Spotify add Missing to x  
  - Edit snapshot to added  
  - count added  
  - Gotify (optional notification)

- **Node Details:**

| Node Name              | Details                                                                                                                  |
|------------------------|-------------------------------------------------------------------------------------------------------------------------|
| Loop add missing        | Split In Batches Node. Role: Loops through missing tracks to add them one by one.                                        |
| Spotify add Missing to x| Spotify API Node. Role: Adds each missing track to the playlist identified by `setpluri` URI.                            |
| Edit snapshot to added  | Set Node. Stores the Spotify snapshot ID after adding tracks, used for tracking changes.                                |
| count added            | Summarize Node. Counts how many tracks were added during this execution.                                                |
| Gotify                 | Gotify Node. Sends a notification summarizing songs added and execution duration. Optional, customizable by user.      |

- **Edge Cases / Failures:**  
  - Adding tracks may fail due to rate limits or invalid track IDs; retry enabled on Spotify add node.  
  - Empty missing tracks list results in no additions.  
  - Gotify notification requires proper configuration and internet access.

---

#### 1.7 Sync - Remove Unliked Songs

- **Overview:**  
  Iterates over tracks in the playlist that are no longer liked and removes them.

- **Nodes Involved:**  
  - Loop delete old  
  - Spotify delete old  
  - Edit success to del  
  - Cound deleted  
  - Gotify Send deleted n from x (optional notification)

- **Node Details:**

| Node Name               | Details                                                                                                                   |
|-------------------------|--------------------------------------------------------------------------------------------------------------------------|
| Loop delete old         | Split In Batches Node. Loops through tracks to be deleted one by one.                                                    |
| Spotify delete old      | Spotify API Node. Deletes each track from the playlist using playlist ID and track URI.                                  |
| Edit success to del     | Set Node. Captures the success state of deletion operations.                                                            |
| Cound deleted           | Summarize Node. Counts how many tracks were deleted.                                                                     |
| Gotify Send deleted n from x | Gotify Node. Sends a notification summarizing deleted tracks and workflow duration. Optional and customizable.          |

- **Edge Cases / Failures:**  
  - Deletion may fail if track is already removed or due to permission issues.  
  - Empty delete list results in no action.  
  - Gotify node requires correct setup.

---

#### 1.8 Summary & Notifications

- **Overview:**  
  Sends notifications via Gotify summarizing the results of the synchronization, including counts of added and deleted songs and total duration.

- **Nodes Involved:**  
  - Gotify  
  - Gotify Send deleted n from x

- **Node Details:**

| Node Name                   | Details                                                                                                                         |
|-----------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| Gotify                      | Sends markdown message summarizing number of songs added, target playlist name, and elapsed time.                              |
| Gotify Send deleted n from x| Sends markdown message summarizing number of songs deleted, target playlist name, and elapsed time.                             |

- **Edge Cases / Failures:**  
  - Gotify service must be configured correctly with valid credentials.  
  - Network or service outages may prevent notifications.

---

### 3. Summary Table

| Node Name                 | Node Type          | Functional Role                                    | Input Node(s)                    | Output Node(s)                     | Sticky Note                                                                                                                        |
|---------------------------|--------------------|---------------------------------------------------|---------------------------------|----------------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| Start                     | Manual Trigger     | Manual execution trigger                           | -                               | Edit set Vars                    |                                                                                                                                   |
| Schedule Trigger          | Schedule Trigger   | Scheduled execution (daily)                        | -                               | Edit set Vars                    | Run the workflow every 24h at 0 o'clock                                                                                           |
| Edit set Vars             | Set                | Set target playlist name variable                  | Start, Schedule Trigger          | Edit set intern vars             | # Edit here!<br>## Change the value to the name of your target playlist.                                                         |
| Edit set intern vars      | Set                | Set internal vars such as start time                | Edit set Vars                   | Spotify get Liked Songs, Spotify get all playlists | # Edit here!<br>### You need to add your own spotify account here.                                                                |
| Spotify get Liked Songs   | Spotify            | Fetch all liked songs                               | Edit set intern vars             | Sort first added to first item   | # Edit here!<br>### You need to add your own spotify account here.                                                                |
| Sort first added to first item | Sort         | Sort liked songs by added date ascending            | Spotify get Liked Songs          | Merge                          |                                                                                                                                   |
| Spotify get all playlists | Spotify            | Fetch all user playlists                            | Edit set intern vars             | Filter Playlist x                | # Edit here!<br>### You need to add your own spotify account here.                                                                |
| Filter Playlist x         | Filter             | Filter playlists by name matching target playlist  | Spotify get all playlists        | Set pluri                      | # Edit here!                                                                                                                      |
| Set pluri                 | Set                | Set playlist URI variable                           | Filter Playlist x                | Spotify get Tracks of X, Merge   | # Edit here!                                                                                                                      |
| Spotify get Tracks of X   | Spotify            | Fetch tracks of target playlist                     | Set pluri                      | Sort                           | # Edit here!<br>### You need to add your own spotify account here.                                                                |
| Sort                      | Sort               | Sort playlist tracks by added date ascending       | Spotify get Tracks of X          | Compare Datasets                |                                                                                                                                   |
| Merge                     | Merge              | Combine liked songs and playlist tracks datasets   | Sort first added to first item, Set pluri | Compare Datasets                | ## Compare the content of your Liked Songs and the target Playlist                                                                |
| Compare Datasets          | Compare Datasets   | Identify missing and extra tracks                    | Sort, Merge                     | Loop add missing, Loop delete old, END |                                                                                                                                   |
| Loop add missing          | Split In Batches   | Loop over missing songs to add                       | Compare Datasets                | Edit snapshot to added, Spotify add Missing to x | ### Spotify add all missing song from your Liked Songs to the Playlist.                                                           |
| Spotify add Missing to x  | Spotify            | Add missing songs to playlist                        | Loop add missing                | Loop add missing               | # Edit here!<br>### You need to add your own spotify account here.                                                                |
| Edit snapshot to added    | Set                | Store snapshot ID after adding tracks               | Loop add missing                | count added                   |                                                                                                                                   |
| count added               | Summarize          | Count how many songs were added                      | Edit snapshot to added          | Gotify                       | ## (Optional) <br>### Count the number of songs that were added                                                                    |
| Gotify                    | Gotify             | Notify added songs summary                           | count added                    | -                             |                                                                                                                                   |
| Loop delete old           | Split In Batches   | Loop over songs to delete                            | Compare Datasets                | Edit success to del, Spotify delete old | ### Spotify remove all songs that aren't in your Liked Songs anymore.                                                              |
| Spotify delete old        | Spotify            | Delete songs from playlist                           | Loop delete old                | Loop delete old               | # Edit here!<br>### You need to add your own spotify account here.                                                                |
| Edit success to del       | Set                | Store deletion success status                        | Loop delete old                | Cound deleted                 |                                                                                                                                   |
| Cound deleted             | Summarize          | Count how many songs were deleted                    | Edit success to del            | Gotify Send deleted n from x  | ## (Optional) <br>### Count the number of songs that were deleted                                                                  |
| Gotify Send deleted n from x | Gotify          | Notify deleted songs summary                         | Cound deleted                 | -                             |                                                                                                                                   |
| END                       | NoOp               | Workflow branch end                                 | Compare Datasets               | -                             |                                                                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Manual Trigger node:**  
   - Name: "Start"  
   - Purpose: Allow manual execution.

3. **Add a Schedule Trigger node:**  
   - Name: "Schedule Trigger"  
   - Configure interval to run every 24 hours at 00:00 (midnight).

4. **Connect both "Start" and "Schedule Trigger" to a Set node:**  
   - Name: "Edit set Vars"  
   - Add an assignment:  
     - Variable: `varplaylistto`  
     - Type: String  
     - Value: Set this to your target playlist name exactly (replace default "CHANGE MEEEEEEEEE").  
   - Add sticky notes nearby to remind users to change this value.

5. **Connect "Edit set Vars" to another Set node:**  
   - Name: "Edit set intern vars"  
   - Add an assignment:  
     - Variable: `timestart`  
     - Type: String  
     - Value: Expression `{{$now.toUnixInteger()}}` to capture execution start time.

6. **From "Edit set intern vars" split output to two Spotify nodes:**

   - **Spotify get Liked Songs:**  
     - Resource: Library  
     - Operation: Get Liked Songs  
     - Return All: true  
     - Use your Spotify credentials here.

   - **Spotify get all playlists:**  
     - Resource: Playlist  
     - Operation: Get User Playlists  
     - Return All: true  
     - Use same Spotify credentials.

7. **Sort Liked Songs by `added_at`:**  
   - Add a Sort node: "Sort first added to first item"  
   - Sort field: `added_at` ascending.

8. **Filter playlists to find target playlist:**  
   - Add Filter node: "Filter Playlist x"  
   - Condition: `$json.name` equals `{{$node["Edit set Vars"].json["varplaylistto"]}}`  
   - Case sensitive, strict match.

9. **Set playlist URI variable:**  
   - Add Set node: "Set pluri"  
   - Assign variable `setpluri` with expression: `{{$json.uri}}`

10. **Fetch tracks from the target playlist:**  
    - Spotify node: "Spotify get Tracks of X"  
    - Resource: Playlist  
    - Operation: Get Tracks  
    - Playlist ID: `{{$node["Set pluri"].json["setpluri"]}}`  
    - Return All: true.

11. **Sort playlist tracks by `added_at`:**  
    - Add Sort node: "Sort"  
    - Sort field: `added_at` ascending.

12. **Merge Liked Songs and Playlist Tracks:**  
    - Add Merge node: Combine mode, multiplex.  
    - Connect "Sort first added to first item" and "Set pluri" outputs to this merge.

13. **Compare datasets:**  
    - Add Compare Datasets node:  
      - Merge by fields: `track.uri` on both datasets.  
      - Outputs: Missing items (to add), Extra items (to delete), matched (ignored).

14. **Add missing songs loop:**  
    - Add SplitInBatches node: "Loop add missing" connected to missing items output.  
    - Add Spotify node: "Spotify add Missing to x"  
      - Operation: Add Track  
      - Playlist ID: `{{$json.setpluri}}`  
      - Track ID: `{{$json.track.uri}}`  
      - Enable retry on fail.  
    - Add Set node: "Edit snapshot to added" to store `snapshot_id` from Spotify response.  
    - Add Summarize node: "count added" to count added tracks.  
    - Optionally add Gotify node to notify additions.

15. **Remove unliked songs loop:**  
    - Add SplitInBatches node: "Loop delete old" connected to extra items output.  
    - Add Spotify node: "Spotify delete old"  
      - Operation: Delete Track  
      - Playlist ID: `{{$json.setpluri}}`  
      - Track ID: `{{$json.track.uri}}`  
    - Add Set node: "Edit success to del" to capture deletion success.  
    - Add Summarize node: "Cound deleted" to count deleted tracks.  
    - Optionally add Gotify node to notify deletions.

16. **Create END nodes for workflow termination after compare dataset outputs.**

17. **Add sticky notes at strategic points:**  
    - Near "Edit set Vars" to instruct changing playlist name.  
    - Near Spotify nodes to remind adding Spotify credentials.  
    - Near Gotify nodes to instruct customization or deletion if not used.

18. **Credentials setup:**  
    - For all Spotify nodes, create and assign Spotify OAuth2 credentials linked to your Spotify Developer app.  
    - For Gotify nodes, create Gotify credentials if notifications are desired.

19. **Test workflow manually and via schedule, monitor logs for errors or failures.**

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                                      |
|-------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| For setting Spotify credentials, visit [Spotify Developer Dashboard](https://developer.spotify.com/dashboard). | Instructions for creating Spotify apps and credentials.                                                             |
| Gotify integration is optional; delete Gotify nodes if not used.                                 | Workflow customization tip.                                                                                          |
| Ensure exact playlist name in "Edit set Vars" node to avoid playlist mismatch errors.             | Critical user setup instruction.                                                                                    |
| Recommended to run the workflow once manually after setup and then rely on scheduled trigger.    | Best practice for initial validation.                                                                                |
| Rate limits on Spotify API may apply; retry is enabled on add operations but monitor for failures.| Error handling note.                                                                                                 |
| Workflow timezone is set to Europe/Berlin; adjust if necessary in workflow settings.             | Scheduling configuration note.                                                                                       |

---

This document provides a full technical understanding of the "Spotify Sync Liked Songs to Playlist" n8n workflow, enabling reproduction, customization, and troubleshooting by advanced users and AI agents.
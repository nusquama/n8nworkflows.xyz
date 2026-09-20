Manage QR-based event reservations with forms, Gmail and Discord

https://n8nworkflows.xyz/workflows/manage-qr-based-event-reservations-with-forms--gmail-and-discord-17699


# Manage QR-based event reservations with forms, Gmail and Discord

### 1. Workflow Overview

This workflow automates the complete event reservation lifecycle, spanning from web form submission and capacity validation to ticket generation, electronic ticket delivery, and real-time dashboard updates on Discord.

The workflow logic is categorized into the following logical blocks:
- **1.1 Input Reception & Normalization:** Receives event booking submissions from an online web form and normalizes user parameters.
- **1.2 Slot Availability & Validation:** Queries available time slots from an n8n Data Table, assesses capacity constraints, automatically reassigns attendees to subsequent valid slots if necessary, and branches execution based on availability.
- **1.3 Rejection Handling:** Handles full-capacity or invalid reservation requests by sending an alert notification to a designated Discord channel.
- **1.4 Reservation Creation & Capacity Update:** Persists valid reservations into a Data Table, updates slot capacities, and invokes a sub-workflow to issue a unique ticket identifier.
- **1.5 E-Ticket Generation & Distribution:** Generates a dynamic QR code using an external API, dispatches the e-ticket to the attendee via Gmail, and finalizes the reservation record with the ticket identifier.
- **1.6 Live Dashboard Synchronization:** Broadcasts booking confirmations to Discord, queries the latest capacity metrics from the data tables, builds a formatted availability markdown board, and updates or creates a pinned overview message via the Discord API.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Normalization
- **Overview:** Captures reservation form inputs from users and standardizes data types for downstream processing.
- **Nodes Involved:** `Reservation Form`, `Normalize Input`
- **Node Details:**
  - `Reservation Form` (`n8n-nodes-base.formTrigger`, v2.6):
    - *Role:* Webhook-based form trigger collecting name, email, slot selection (`SLOT-A`, `SLOT-B`, `SLOT-C`), and party size.
    - *Expressions/Variables:* Exposes incoming fields via `$json`.
    - *Connections:* Input (None - Trigger); Output (`Normalize Input`).
    - *Edge Cases/Failures:* Form submission network drops, invalid input types.
  - `Normalize Input` (`n8n-nodes-base.set`, v3.4):
    - *Role:* Maps and casts incoming form fields to expected string and number formats.
    - *Configuration:* Assigns `name` (string), `email` (string), `slot_id` (string), and `party_size` (numeric cast using `Number()`).
    - *Connections:* Input (`Reservation Form`); Output (`time_slots_v2`).
    - *Edge Cases/Failures:* Expression execution failures if incoming fields are missing or null.

#### 2.2 Slot Availability & Validation
- **Overview:** Retrieves data table records for time slots, evaluates requested capacity against live counts, and branches the workflow.
- **Nodes Involved:** `time_slots_v2`, `Find Available Slot`, `Slot Available?`
- **Node Details:**
  - `time_slots_v2` (`n8n-nodes-base.dataTable`, v1.1):
    - *Role:* Fetches all configured time slots from the `time_slots_v2` Data Table.
    - *Configuration:* Operation: `get`, return all rows, Table ID: `J4lhpHmHfFHJg5Zq`.
    - *Connections:* Input (`Normalize Input`); Output (`Find Available Slot`).
    - *Edge Cases/Failures:* Data table connection errors or unpopulated tables causing empty arrays.
  - `Find Available Slot` (`n8n-nodes-base.code`, v2.0):
    - *Role:* JavaScript execution block that evaluates requested slot availability, validates party size constraints (1–5 people), and automatically finds the next open slot if the requested one is full.
    - *Expressions/Variables:* Reads data from `Normalize Input` and items from `time_slots_v2`. Outputs structured availability flags (`available`, `reassigned`, `slot_status`, `assigned_slot_id`, etc.).
    - *Connections:* Input (`time_slots_v2`); Output (`Slot Available?`).
    - *Edge Cases/Failures:* Unhandled exceptions if slot order properties contain non-numeric characters.
  - `Slot Available?` (`n8n-nodes-base.if`, v2.3):
    - *Role:* Conditional router routing valid reservations forward or rejecting over-capacity requests.
    - *Configuration:* Evaluates `{{ $json.available }}` equals `true`.
    - *Connections:* Input (`Find Available Slot`); Outputs (`Create Reservation` on true, `Send Full Capacity Notification` on false).
    - *Edge Cases/Failures:* Type mismatch on boolean evaluation.

#### 2.3 Rejection Handling
- **Overview:** Notifies administrators via Discord when a reservation cannot be fulfilled.
- **Nodes Involved:** `Send Full Capacity Notification`
- **Node Details:**
  - `Send Full Capacity Notification` (`n8n-nodes-base.discord`, v2.0):
    - *Role:* Posts rejection details to a specific Discord operations channel.
    - *Configuration:* Resource: `message`, Guild ID: `1531972015546958004`, Channel ID: `1531972016465383488`, Message formatted with name, requested slot, party size, and rejection reason.
    - *Connections:* Input (`Slot Available?` - False branch); Output (None - Terminal).
    - *Edge Cases/Failures:* Discord API rate limits, invalid bot tokens, or channel permissions errors.

#### 2.4 Reservation Creation & Capacity Update
- **Overview:** Records confirmed bookings in the database, increments slot reserved counts, and triggers ticket generation.
- **Nodes Involved:** `Create Reservation`, `Update Slot Capacity`, `Generate Ticket`, `Update row(s)1`
- **Node Details:**
  - `Create Reservation` (`n8n-nodes-base.dataTable`, v1.1):
    - *Role:* Inserts a new row into the `予約者` (Reservations) Data Table.
    - *Configuration:* Maps `name`, `email`, `slot_id`, `party_size`, and sets `reserved_at` using `{{ $now.toISO() }}`. Table ID: `oQSxnyJA0fgv9GEm`.
    - *Connections:* Input (`Slot Available?` - True branch); Output (`Update Slot Capacity`).
    - *Edge Cases/Failures:* Table write constraints or duplicate primary key conflicts.
  - `Update Slot Capacity` (`n8n-nodes-base.dataTable`, v1.1):
    - *Role:* Updates reserved counts and status (`open` or `closed`) in the `time_slots_v2` Data Table.
    - *Configuration:* Operation: `update`, Filter: `slot_id` matches `assigned_slot_id`, updates `reserved_count` and `status`.
    - *Connections:* Input (`Create Reservation`); Output (`Generate Ticket`).
    - *Edge Cases/Failures:* Filter mismatch failing to target the correct slot row.
  - `Generate Ticket` (`n8n-nodes-base.executeWorkflow`, v1.3):
    - *Role:* Executes a child workflow (`チケット発行`) to generate a unique ticket identifier.
    - *Configuration:* Passes `name`, `email`, and `slot_id` as inputs to Workflow ID `5QRvO7RKI0HzHnv2`.
    - *Connections:* Input (`Update Slot Capacity`); Output (`Update row(s)1`).
    - *Sub-Workflow Reference:* Invokes sub-workflow ID `5QRvO7RKI0HzHnv2`.
    - *Edge Cases/Failures:* Sub-workflow execution failure or timeout.
  - `Update row(s)1` (`n8n-nodes-base.dataTable`, v1.1):
    - *Role:* Updates the reservation record with the newly generated ticket identifier.
    - *Configuration:* Operation: `update`, Filter: `id` matches `Create Reservation` record ID, updates `ticket_id`. Table ID: `oQSxnyJA0fgv9GEm`.
    - *Connections:* Input (`Generate Ticket`); Output (`Generate QR Code`).
    - *Edge Cases/Failures:* Data table row identifier mismatch.

#### 2.5 E-Ticket Generation & Distribution
- **Overview:** Creates a QR code image representing the ticket ID and delivers it to the user via Gmail.
- **Nodes Involved:** `Generate QR Code`, `Send E-Ticket`, `Update row(s)`
- **Node Details:**
  - `Generate QR Code` (`n8n-nodes-base.httpRequest`, v4.4):
    - *Role:* Calls the QuickChart API to generate a PNG QR code image based on the ticket ID.
    - *Configuration:* GET request to `https://quickchart.io/qr?text=...&size=350&format=png&margin=3&ecLevel=M`, binary response output assigned to property `qr_ticket`.
    - *Connections:* Input (`Update row(s)1`); Output (`Send E-Ticket`).
    - *Edge Cases/Failures:* QuickChart API downtime or network timeout.
  - `Send E-Ticket` (`n8n-nodes-base.gmail`, v2.2):
    - *Role:* Sends a confirmation HTML email with ticket details and the QR code binary attachment via Gmail.
    - *Configuration:* Recipient: attendee email, Subject includes ticket ID, Attachments binary: `qr_ticket`.
    - *Connections:* Input (`Generate QR Code`); Output (`Update row(s)`); Credentials: Gmail OAuth2 / API.
    - *Edge Cases/Failures:* Invalid email format, SMTP limits, or authentication revocation.
  - `Update row(s)` (`n8n-nodes-base.dataTable`, v1.1):
    - *Role:* Performs an auxiliary update on the reservation record table.
    - *Configuration:* Operation: `update`, Table ID: `oQSxnyJA0fgv9GEm`.
    - *Connections:* Input (`Send E-Ticket`); Output (`Send Reservation Notification`).
    - *Edge Cases/Failures:* Database update error.

#### 2.6 Live Dashboard Synchronization
- **Overview:** Notifies administrators of successful bookings in Discord and maintains a continuously updated availability summary message.
- **Nodes Involved:** `Send Reservation Notification`, `Reload Time Slots`, `Build Availability Message`, `Get Message Settings`, `Message Exists?`, `Update Discord Message`, `Send a message`, `Save Message ID`
- **Node Details:**
  - `Send Reservation Notification` (`n8n-nodes-base.discord`, v2.0):
    - *Role:* Posts a booking success notification to the administrator Discord channel.
    - *Configuration:* Channel ID: `1531972016465383487`.
    - *Connections:* Input (`Update row(s)`); Output (`Reload Time Slots`).
  - `Reload Time Slots` (`n8n-nodes-base.dataTable`, v1.1):
    - *Role:* Re-fetches the latest slot data to ensure availability board accuracy.
    - *Configuration:* Operation: `get`, return all rows, Table ID: `J4lhpHmHfFHJg5Zq`.
    - *Connections:* Input (`Send Reservation Notification`); Output (`Build Availability Message`).
  - `Build Availability Message` (`n8n-nodes-base.code`, v2.0):
    - *Role:* JavaScript code block that compiles slot capacities into a structured Markdown availability board with timestamps.
    - *Connections:* Input (`Reload Time Slots`); Output (`Get Message Settings`).
  - `Get Message Settings` (`n8n-nodes-base.dataTable`, v1.1):
    - *Role:* Retrieves configuration settings storing the pinned Discord availability message ID.
    - *Configuration:* Filter: `setting_key` equals `availability_message`, Table ID: `UdxZ9jqTxJeK6Uol`.
    - *Connections:* Input (`Build Availability Message`); Output (`Message Exists?`).
  - `Message Exists?` (`n8n-nodes-base.if`, v2.3):
    - *Role:* Checks whether a Discord availability message ID has previously been saved.
    - *Connections:* Input (`Get Message Settings`); Outputs (`Update Discord Message` if message exists, `Send a message` if empty).
  - `Update Discord Message` (`n8n-nodes-base.httpRequest`, v4.4):
    - *Role:* Updates an existing Discord message via the Discord REST API PATCH method using an HTTP Header Auth bot token.
    - *Credentials:* HTTP Header Auth (`Authorization`).
    - *Connections:* Input (`Message Exists?` - True branch); Output (None - Terminal).
  - `Send a message` (`n8n-nodes-base.discord`, v2.0):
    - *Role:* Sends a brand-new availability message if no previous message ID was found.
    - *Channel ID:* `1531972016465383489`.
    - *Connections:* Input (`Message Exists?` - False branch); Output (`Save Message ID`).
  - `Save Message ID` (`n8n-nodes-base.dataTable`, v1.1):
    - *Role:* Saves the newly created Discord message ID into the `workflow_settings` table for future updates.
    - *Configuration:* Operation: `update`, Table ID: `UdxZ9jqTxJeK6Uol`.
    - *Connections:* Input (`Send a message`); Output (None - Terminal).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `time_slots_v2` | `n8n-nodes-base.dataTable` | Retrieves current time slot information from the Data Table, including capacity, reservation count, and availability status. | `Normalize Input` | `Find Available Slot` | time_slots_v2<br><br>Retrieves the current time slot information from the Data Table, including capacity, reservation count, and availability status. |
| `Send a message` | `n8n-nodes-base.discord` | Sends a new Discord message containing the availability summary. | `Message Exists?` | `Save Message ID` | Update Discord Message / Send a Message<br><br>Updates the existing Discord message or creates a new one. |
| `Update row(s)` | `n8n-nodes-base.dataTable` | Updates the reservation record by saving the generated ticket ID and other reservation details in the Data Table. | `Send E-Ticket` | `Send Reservation Notification` | Update row(s)<br><br>Updates the reservation record by saving the generated ticket ID and other reservation details in the Data Table. |
| `Update row(s)1` | `n8n-nodes-base.dataTable` | Updates the reservation record with the generated ticket ID. | `Generate Ticket` | `Generate QR Code` | Update Discord Message / Send a Message<br><br>Updates the existing Discord message or creates a new one. |
| `Reservation Form` | `n8n-nodes-base.formTrigger` | Receives event reservations via web form. | None (Trigger) | `Normalize Input` | Reservation Form<br><br>Receives reservation requests from participants. |
| `Normalize Input` | `n8n-nodes-base.set` | Formats and validates submitted form data. | `Reservation Form` | `time_slots_v2` | Normalize Input<br><br>Formats and validates the submitted form data. |
| `Find Available Slot` | `n8n-nodes-base.code` | Checks availability and automatically assigns the next available slot if necessary. | `time_slots_v2` | `Slot Available?` | Find Available Slot<br><br>Checks availability and automatically assigns the next available slot if necessary. |
| `Slot Available?` | `n8n-nodes-base.if` | Determines whether a reservation can be accepted. | `Find Available Slot` | `Create Reservation`, `Send Full Capacity Notification` | Slot Available?<br><br>Determines whether a reservation can be accepted. |
| `Create Reservation` | `n8n-nodes-base.dataTable` | Stores participant information in the reservation database. | `Slot Available?` | `Update Slot Capacity` | Create Reservation<br><br>Stores participant information in the reservation database. |
| `Update Slot Capacity` | `n8n-nodes-base.dataTable` | Updates the remaining capacity for the assigned time slot. | `Create Reservation` | `Generate Ticket` | Update Slot Capacity<br><br>Updates the remaining capacity for the assigned time slot. |
| `Generate Ticket` | `n8n-nodes-base.executeWorkflow` | Creates a unique ticket ID for the reservation via sub-workflow. | `Update Slot Capacity` | `Update row(s)1` | Generate Ticket<br><br>Creates a unique ticket ID for the reservation. |
| `Generate QR Code` | `n8n-nodes-base.httpRequest` | Creates a QR code containing the ticket ID. | `Update row(s)1` | `Send E-Ticket` | Generate QR Code<br><br>Creates a QR code containing the ticket ID. |
| `Send E-Ticket` | `n8n-nodes-base.gmail` | Sends the participant an email containing their QR ticket. | `Generate QR Code` | `Update row(s)` | Send E-Ticket<br><br>Sends the participant an email containing their QR ticket. |
| `Send Reservation Notification` | `n8n-nodes-base.discord` | Notifies administrators in Discord about successful bookings. | `Update row(s)` | `Reload Time Slots` | Send Reservation Notification<br><br>Notifies administrators in Discord. |
| `Reload Time Slots` | `n8n-nodes-base.dataTable` | Retrieves the latest slot information. | `Send Reservation Notification` | `Build Availability Message` | Reload Time Slots<br><br>Retrieves the latest slot information. |
| `Build Availability Message` | `n8n-nodes-base.code` | Builds a live availability summary. | `Reload Time Slots` | `Get Message Settings` | Build Availability Message<br><br>Builds a live availability summary. |
| `Get Message Settings` | `n8n-nodes-base.dataTable` | Retrieves the stored Discord message ID. | `Build Availability Message` | `Message Exists?` | Get Message Settings<br><br>Retrieves the stored Discord message ID. |
| `Message Exists?` | `n8n-nodes-base.if` | Checks whether the availability message already exists. | `Get Message Settings` | `Update Discord Message`, `Send a message` | Message Exists?<br><br>Checks whether the availability message already exists. |
| `Update Discord Message` | `n8n-nodes-base.httpRequest` | Updates the existing Discord message. | `Message Exists?` | None (Terminal) | Update Discord Message / Send a Message<br><br>Updates the existing Discord message or creates a new one. |
| `Save Message ID` | `n8n-nodes-base.dataTable` | Stores the Discord message ID in the Data Table so the same message can be updated. | `Send a message` | None (Terminal) | Save Message ID<br><br>Stores the Discord message ID in the Data Table so the same message can be updated instead of creating duplicate messages. |
| `Send Full Capacity Notification` | `n8n-nodes-base.discord` | Notifies administrators through Discord when all available time slots are fully booked. | `Slot Available?` | None (Terminal) | Send Full Capacity Notification<br><br>Notifies administrators through Discord when all available time slots are fully booked and no further reservations can be accepted. |
| `Send Full Capacity Notification` | `n8n-nodes-base.discord` | Notifies administrators when no reservation can be accepted. | `Slot Available?` | None (Terminal) | Send Full Capacity Notification<br><br>Notifies administrators when no reservation can be accepted. |

---

### 4. Reproducing the Workflow from Scratch

To rebuild this workflow manually in n8n, follow these sequential steps:

1. **Create Data Tables:**
   - **Time Slots Table (`time_slots_v2`, ID: `J4lhpHmHfFHJg5Zq`):** Columns: `slot_id` (string), `slot_label` (string), `slot_order` (number), `capacity` (number), `reserved_count` (number), `status` (string).
   - **Reservations Table (`予約者`, ID: `oQSxnyJA0fgv9GEm`):** Columns: `name` (string), `email` (string), `slot_id` (string), `party_size` (number), `ticket_id` (string), `reserved_at` (dateTime).
   - **Workflow Settings Table (`workflow_settings`, ID: `UdxZ9jqTxJeK6Uol`):** Columns: `setting_key` (string), `setting_value` (string), `channel_id` (string).

2. **Add Form Trigger Node:**
   - Type: `n8n-nodes-base.formTrigger` (Version 2.6)
   - Configuration: Path: `event-reservation`, Form Title: `イベント予約フォーム`. Add form fields: `name` (string, required), `email` (email, required), `slot_id` (dropdown with options `SLOT-A`, `SLOT-B`, `SLOT-C`, required), `party_size` (number, required).

3. **Add Input Normalization Node:**
   - Type: `n8n-nodes-base.set` (Version 3.4)
   - Configuration: Assign `name` (`={{ $json.name }}`), `email` (`={{ $json.email }}`), `slot_id` (`={{ $json.slot_id }}`), `party_size` (`={{ Number($json.party_size) }}`).
   - Connection: `Reservation Form` ➔ `Normalize Input`.

4. **Add Time Slots Retrieval Node:**
   - Type: `n8n-nodes-base.dataTable` (Version 1.1)
   - Configuration: Operation: `get`, Return All: true, Data Table: `time_slots_v2`.
   - Connection: `Normalize Input` ➔ `time_slots_v2`.

5. **Add Slot Validation Code Node:**
   - Type: `n8n-nodes-base.code` (Version 2.0)
   - Configuration: Insert custom JavaScript logic to validate requested slot capacity, check party limits (1–5), and determine fallback open slots.
   - Connection: `time_slots_v2` ➔ `Find Available Slot`.

6. **Add Availability Conditional Node:**
   - Type: `n8n-nodes-base.if` (Version 2.3)
   - Configuration: Condition: `{{ $json.available }}` equals `true`.
   - Connection: `Find Available Slot` ➔ `Slot Available?`.

7. **Build Rejection Branch (False path):**
   - Add a Discord node (`n8n-nodes-base.discord`, Version 2.0) named `Send Full Capacity Notification`.
   - Configuration: Resource: `message`, Guild ID, Channel ID, message content incorporating failure details (`{{ $json.reason }}`).
   - Connection: `Slot Available?` (False) ➔ `Send Full Capacity Notification`.

8. **Build Success Branch (True path):**
   - **Create Reservation Node:** Type `n8n-nodes-base.dataTable` (Version 1.1). Insert row into Reservations table mapping form data and setting `reserved_at` to `{{ $now.toISO() }}`.
     - Connection: `Slot Available?` (True) ➔ `Create Reservation`.
   - **Update Slot Capacity Node:** Type `n8n-nodes-base.dataTable` (Version 1.1). Update `time_slots_v2` setting `reserved_count` and `status` based on validation output.
     - Connection: `Create Reservation` ➔ `Update Slot Capacity`.
   - **Generate Ticket Sub-Workflow Node:** Type `n8n-nodes-base.executeWorkflow` (Version 1.3). Reference Workflow ID `5QRvO7RKI0HzHnv2` (チケット発行), mapping input parameters (`name`, `email`, `slot_id`).
     - Connection: `Update Slot Capacity` ➔ `Generate Ticket`.
   - **Update Reservation Ticket ID Node:** Type `n8n-nodes-base.dataTable` (Version 1.1). Update reservation record with generated `ticket_id`.
     - Connection: `Generate Ticket` ➔ `Update row(s)1`.
   - **Generate QR Code HTTP Request Node:** Type `n8n-nodes-base.httpRequest` (Version 4.4). GET request to QuickChart URL encoding `ticket_id`. Set response format to File with output property name `qr_ticket`.
     - Connection: `Update row(s)1` ➔ `Generate QR Code`.
   - **Send E-Ticket Gmail Node:** Type `n8n-nodes-base.gmail` (Version 2.2). Configure Gmail credentials, recipient email, subject, HTML message body, and attach binary property `qr_ticket`.
     - Connection: `Generate QR Code` ➔ `Send E-Ticket`.
   - **Auxiliary Reservation Update Node:** Type `n8n-nodes-base.dataTable` (Version 1.1).
     - Connection: `Send E-Ticket` ➔ `Update row(s)`.
   - **Send Discord Notification Node:** Type `n8n-nodes-base.discord` (Version 2.0). Post reservation confirmation to administrator channel.
     - Connection: `Update row(s)` ➔ `Send Reservation Notification`.

9. **Build Live Availability Dashboard Sub-Flow:**
   - **Reload Time Slots Node:** Type `n8n-nodes-base.dataTable` (Version 1.1). Fetch all rows from `time_slots_v2`.
     - Connection: `Send Reservation Notification` ➔ `Reload Time Slots`.
   - **Build Availability Message Code Node:** Type `n8n-nodes-base.code` (Version 2.0). Format slot capacities into Markdown text.
     - Connection: `Reload Time Slots` ➔ `Build Availability Message`.
   - **Get Message Settings Node:** Type `n8n-nodes-base.dataTable` (Version 1.1). Get row from `workflow_settings` where `setting_key` equals `availability_message`.
     - Connection: `Build Availability Message` ➔ `Get Message Settings`.
   - **Message Exists? Conditional Node:** Type `n8n-nodes-base.if` (Version 2.3). Check if `setting_value` is not empty.
     - Connection: `Get Message Settings` ➔ `Message Exists?`.
   - **Update Discord Message (True path):** Type `n8n-nodes-base.httpRequest` (Version 4.4). PATCH request to Discord API using HTTP Header Auth credentials (`ngmBUpkmFKswko23`).
     - Connection: `Message Exists?` (True) ➔ `Update Discord Message`.
   - **Send New Message (False path):** Type `n8n-nodes-base.discord` (Version 2.0). Send message to Discord availability channel.
     - Connection: `Message Exists?` (False) ➔ `Send a message`.
   - **Save Message ID Node:** Type `n8n-nodes-base.dataTable` (Version 1.1). Update `workflow_settings` with the new message ID.
     - Connection: `Send a message` ➔ `Save Message ID`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Target Use Cases: Event reservations, university workshops, conferences, open campus events, and community gatherings. | Japanese event management workflow design |
| Required Credentials: Gmail OAuth2, Discord Bot integration, and HTTP Header Auth (Discord Bot Token). | External API Authentication |
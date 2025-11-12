Automated Restaurant Call Handling & Table Booking System with VAPI and PostgreSQL

https://n8nworkflows.xyz/workflows/automated-restaurant-call-handling---table-booking-system-with-vapi-and-postgresql-5466


# Automated Restaurant Call Handling & Table Booking System with VAPI and PostgreSQL

### 1. Workflow Overview

This workflow automates the handling of restaurant phone calls via a Virtual API (VAPI), enabling customers to check table availability and book tables seamlessly. It integrates VAPI for conversational call interactions, PostgreSQL for managing table data, and n8n’s webhook and database nodes for orchestration.

The workflow is logically divided into two main blocks:

- **1.1 Availability Check Flow:** Receives requests via VAPI to check if tables are available for a specified time and party size. It queries the PostgreSQL database and responds back to VAPI with availability status.

- **1.2 Booking Flow:** Handles booking requests from VAPI, inserts or updates booking information in PostgreSQL, and sends booking confirmation back to VAPI.

Both blocks operate independently triggered by separate webhook endpoints but share the common goal of facilitating smooth restaurant call handling and table booking.

---

### 2. Block-by-Block Analysis

#### 2.1 Availability Check Flow

- **Overview:**  
  This block listens for incoming availability check requests via a VAPI webhook, queries the PostgreSQL database for table availability, and responds to the caller with the availability status.

- **Nodes Involved:**  
  - Trigger: Booking Request (VAPI)  
  - Query Table Availability (Postgres)  
  - Respond: Availability Status (VAPI)  
  - Sticky Notes (contextual descriptions)

- **Node Details:**

  1. **Trigger: Booking Request (VAPI)**  
     - Type: Webhook  
     - Role: Entry point for availability check requests from VAPI (HTTP POST).  
     - Configuration:  
       - Webhook path: `027f0f14-93f4-42ff-90a7-715f23316a86`  
       - HTTP Method: POST  
       - Response Mode: Waits for response node to reply.  
     - Inputs: External HTTP POST from VAPI.  
     - Outputs: Passes JSON payload to next node.  
     - Edge Cases:  
       - Invalid or missing request body could cause failures.  
       - Network or authentication issues on webhook endpoint.  
     - Version: 2  

  2. **Query Table Availability (Postgres)**  
     - Type: PostgreSQL Node  
     - Role: Queries the database to check if a table matching the criteria is available.  
     - Configuration:  
       - Operation: Select from table `table_id` in schema `public`.  
       - Query parameters are dynamically mapped from webhook input (not explicitly shown but implied).  
       - Uses PostgreSQL credentials stored securely.  
     - Inputs: JSON data from webhook with booking criteria (e.g., date/time, party size).  
     - Outputs: Query results indicating availability.  
     - Edge Cases:  
       - Database connection failures, query errors, or empty results.  
       - Data type mismatches in query parameters.  
     - Version: 2.6  

  3. **Respond: Availability Status (VAPI)**  
     - Type: Respond to Webhook  
     - Role: Sends JSON response back to VAPI with availability status.  
     - Configuration:  
       - Responds with JSON body containing `toolCallId` extracted from incoming webhook payload and `result` set to the availability status from the query.  
       - Uses expressions to access nested JSON data for toolCallId.  
     - Inputs: Output from PostgreSQL query node.  
     - Outputs: HTTP response to VAPI caller.  
     - Edge Cases:  
       - Expression errors if expected JSON path is missing.  
       - Timeout if response is delayed in workflow.  
     - Version: 1.2  

#### 2.2 Booking Flow

- **Overview:**  
  This block processes booking requests from VAPI, updates/inserts booking details into the PostgreSQL database, and sends back booking confirmations.

- **Nodes Involved:**  
  - Trigger: Booking Request (VAPI) 1  
  - Upsert Booking in Postgres  
  - Respond: Booking Confirmation (VAPI)  
  - Sticky Notes (contextual descriptions)

- **Node Details:**

  1. **Trigger: Booking Request (VAPI) 1**  
     - Type: Webhook  
     - Role: Entry point for booking requests from VAPI (HTTP POST).  
     - Configuration:  
       - Webhook path: `2f7eff83-2e85-45ee-b544-7f889ca3ad07`  
       - HTTP Method: POST  
       - Response Mode: Waits for response node to reply.  
     - Inputs: Booking request data from VAPI.  
     - Outputs: Passes JSON payload to next node.  
     - Edge Cases:  
       - Invalid or incomplete booking data.  
       - Network or webhook errors.  
     - Version: 2  

  2. **Upsert Booking in Postgres**  
     - Type: PostgreSQL Node  
     - Role: Inserts or updates booking information into the `table_id` table.  
     - Configuration:  
       - Operation: Upsert (insert or update) on table `table_id` in schema `public`.  
       - Columns are automatically mapped from input data.  
       - Uses PostgreSQL credentials.  
     - Inputs: Booking data from webhook.  
     - Outputs: Status of database operation (success/failure).  
     - Edge Cases:  
       - Database connectivity issues.  
       - Upsert conflicts or schema mismatches.  
       - Data validation failures.  
     - Version: 2.6  

  3. **Respond: Booking Confirmation (VAPI)**  
     - Type: Respond to Webhook  
     - Role: Sends booking confirmation back to VAPI with operation status.  
     - Configuration:  
       - Responds with JSON containing `toolCallId` from original webhook payload and the `status` of the booking operation.  
       - Uses expressions to extract nested JSON data.  
     - Inputs: Output of upsert operation.  
     - Outputs: HTTP response to VAPI caller.  
     - Edge Cases:  
       - Expression failures if JSON path is invalid.  
       - Timeout or communication issues.  
     - Version: 1.2  

---

### 3. Summary Table

| Node Name                      | Node Type            | Functional Role                        | Input Node(s)                | Output Node(s)                  | Sticky Note                                                                                   |
|-------------------------------|----------------------|-------------------------------------|-----------------------------|--------------------------------|----------------------------------------------------------------------------------------------|
| Sticky Note                   | Sticky Note          | Workflow description and context     |                             |                                | 📝 Description: Handles incoming calls for the restaurant using VAPI, PostgreSQL, and n8n.   |
| Sticky Note1                  | Sticky Note          | Availability Check Flow label         |                             |                                | 🟢 Availability Check Flow                                                                    |
| Trigger: Booking Request (VAPI) | Webhook              | Entry point for availability check   |                             | Query Table Availability (Postgres) |                                                                                              |
| Query Table Availability (Postgres) | PostgreSQL           | Checks table availability in DB      | Trigger: Booking Request (VAPI) | Respond: Availability Status (VAPI) |                                                                                              |
| Respond: Availability Status (VAPI) | Respond to Webhook    | Sends availability status to VAPI    | Query Table Availability (Postgres) |                                |                                                                                              |
| Sticky Note3                  | Sticky Note          | Booking Flow label                    |                             |                                | 🔵 Booking Flow                                                                              |
| Trigger: Booking Request (VAPI) 1 | Webhook              | Entry point for booking requests      |                             | Upsert Booking in Postgres     |                                                                                              |
| Upsert Booking in Postgres    | PostgreSQL           | Inserts or updates booking in DB     | Trigger: Booking Request (VAPI) 1 | Respond: Booking Confirmation (VAPI) |                                                                                              |
| Respond: Booking Confirmation (VAPI) | Respond to Webhook    | Sends booking confirmation to VAPI   | Upsert Booking in Postgres   |                                |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Workflow:** Name it "Restaurant Virtual Receptionist & Table Booking with VAPI and n8n".

2. **Add Sticky Notes:**  
   - Add a sticky note with the workflow description.  
   - Add sticky notes labeling the "Availability Check Flow" and "Booking Flow" areas for clarity.

3. **Availability Check Flow Setup:**

   a. **Add Webhook Node:**  
      - Name: `Trigger: Booking Request (VAPI)`  
      - Type: Webhook (HTTP POST)  
      - Webhook path: `027f0f14-93f4-42ff-90a7-715f23316a86`  
      - Response Mode: `Response Node` (waits for response node to reply)  

   b. **Add PostgreSQL Node:**  
      - Name: `Query Table Availability (Postgres)`  
      - Type: PostgreSQL  
      - Operation: `Select`  
      - Table: `table_id`  
      - Schema: `public`  
      - Configure columns/filters to match availability criteria (time, party size) — map these from webhook input JSON accordingly.  
      - Credentials: Select or create PostgreSQL credentials connected to your DB instance.

   c. **Connect:** `Trigger: Booking Request (VAPI)` → `Query Table Availability (Postgres)`

   d. **Add Respond to Webhook Node:**  
      - Name: `Respond: Availability Status (VAPI)`  
      - Respond With: JSON  
      - Response Body:  
        ```json
        {
          "results": [
            {
              "toolCallId": "={{ $('Trigger: Booking Request (VAPI)').item.json.body.message.toolCalls[0].id }}",
              "result": "{{ $json.available }}"
            }
          ]
        }
        ```  
      - Connect: `Query Table Availability (Postgres)` → `Respond: Availability Status (VAPI)`

4. **Booking Flow Setup:**

   a. **Add Webhook Node:**  
      - Name: `Trigger: Booking Request (VAPI) 1`  
      - Type: Webhook (HTTP POST)  
      - Webhook path: `2f7eff83-2e85-45ee-b544-7f889ca3ad07`  
      - Response Mode: `Response Node`

   b. **Add PostgreSQL Node:**  
      - Name: `Upsert Booking in Postgres`  
      - Type: PostgreSQL  
      - Operation: `Upsert`  
      - Table: `table_id`  
      - Schema: `public`  
      - Columns: Auto-map from incoming webhook data to booking fields (e.g., name, booking time, number of people).  
      - Credentials: Use PostgreSQL credentials.

   c. **Connect:** `Trigger: Booking Request (VAPI) 1` → `Upsert Booking in Postgres`

   d. **Add Respond to Webhook Node:**  
      - Name: `Respond: Booking Confirmation (VAPI)`  
      - Respond With: JSON  
      - Response Body:  
        ```json
        {
          "results": [
            {
              "toolCallId": "={{ $('Trigger: Booking Request (VAPI) 1').item.json.body.message.toolCalls[0].id }}",
              "result": "{{ $json.status }}"
            }
          ]
        }
        ```  
      - Connect: `Upsert Booking in Postgres` → `Respond: Booking Confirmation (VAPI)`

5. **Set Workflow Settings:**  
   - Timezone: `Asia/Kolkata` (adjust as necessary)  
   - Execution Order: `v1` (sequential)  
   - Caller Policy: `workflowsFromSameOwner` (restrict calls to same owner workflows)

6. **Credentials:**  
   - PostgreSQL: Create credentials with proper host, port, user, password, and database name.  
   - VAPI: No explicit credential needed here as VAPI connects via webhook. Ensure your VAPI integration is configured to call the webhook URLs.

7. **Activate Workflow:** Test both webhook endpoints for availability checks and bookings. Monitor logs for errors or misconfigurations.

---

### 5. General Notes & Resources

| Note Content                                                                                 | Context or Link                                  |
|----------------------------------------------------------------------------------------------|-------------------------------------------------|
| This workflow leverages VAPI webhooks to handle restaurant call automation and table booking. | Use VAPI documentation to configure webhook calls properly. |
| PostgreSQL node requires proper schema and table `table_id` with fields for bookings.         | Ensure your database schema matches the workflow's expected columns. |
| The workflow uses expressions to extract `toolCallId` from VAPI payloads for response tracking. | Useful for correlating requests and responses in VAPI logging. |
| Timezone is set to Asia/Kolkata; adjust if deploying elsewhere.                              | Workflow setting for correct timestamp handling. |
| Workflow includes important sticky notes for visual guidance in the n8n editor interface.    | Helps maintain clarity and documentation within the editor. |

---

**Disclaimer:**  
The text provided comes exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.
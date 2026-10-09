Book confirmed webhook appointments in Google Calendar using a POST endpoint

https://n8nworkflows.xyz/workflows/book-confirmed-webhook-appointments-in-google-calendar-using-a-post-endpoint-20494


# Book confirmed webhook appointments in Google Calendar using a POST endpoint

### 1. Workflow Overview

The **“Book confirmed appointments in Google Calendar”** workflow acts as a secure, authenticated POST webhook endpoint designed to convert pre-confirmed appointment payloads into private Google Calendar events. It targets teams integrating external booking forms, virtual assistants, or internal administrative tools with Google Calendar without handling the front-end user confirmation flow.

The processing logic is structured into the following functional blocks:
- **1.1 Input Reception & Validation:** Captures incoming HTTP POST requests, enforces authentication, and runs rigorous synthetic validation on payload fields (RFC3339 timestamps, duration limits, future constraints, summary length, and base32hex request ID format). Invalid payloads return an HTTP 400 status.
- **1.2 Calendar Availability Check:** Queries the target Google Calendar for slot availability across the requested timeframe. Provider errors result in an HTTP 502 response, while busy slots trigger an HTTP 409 response.
- **1.3 Event Creation & Post-Creation Verification:** Creates a private, non-attendee calendar event using the stable `request_id` as the Google Calendar event ID. It performs strict structural checks on the API receipt (validating ID matching, confirmed status, and precise timestamp alignment) before responding with HTTP 201 or handling failures with an HTTP 502 error.

---

### 2. Block-by-Block Analysis

---

### Block 1.1: Input Reception & Validation

#### Overview
This block receives inbound HTTP traffic via a webhook, enforces header-based authentication, and validates all required appointment metadata using custom JavaScript to protect the calendar from malformed or unconfirmed submissions.

#### Nodes Involved
- `Receive Request`
- `Validate Request`
- `Request Is Valid`
- `Return Invalid`

#### Node Details

##### **Receive Request**
- **Type & Technical Role:** `n8n-nodes-base.webhook` (v2) — Entry point exposing an HTTP POST webhook endpoint.
- **Configuration Choices:** Configured for `POST` method, uses `headerAuth` (Webhook Header Auth), responds via the `responseNode` mode on a dedicated path (`template-calendar-booking`).
- **Key Expressions / Variables:** Endpoint path: `template-calendar-booking`.
- **Input / Output Connections:** Input: External HTTP caller $\rightarrow$ Output: `Validate Request`.
- **Version-Specific Requirements:** Requires a Webhook Header Auth credential setup in n8n.
- **Edge Cases / Failure Types:** Unauthorized callers are rejected at the webhook level if header credentials do not match.

##### **Validate Request**
- **Type & Technical Role:** `n8n-nodes-base.code` (v2) — Executes a custom JavaScript parsing routine to validate the incoming JSON body.
- **Configuration Choices:** Extracts `start`, `end`, `confirmed`, `summary`, and `request_id`. Validates that timestamps conform to strict RFC3339 formats with seconds and explicit offsets, ensures start is in the future, checks that the duration is $\le$ 1 day, validates `confirmed === true`, checks summary length ($\le$ 160 characters), and tests `request_id` against a lowercase base32hex regex (`/^[a-v0-9]{16,64}$/`).
- **Key Expressions / Variables:** Accesses `$input.first().json.body`.
- **Input / Output Connections:** Input: `Receive Request` $\rightarrow$ Output: `Request Is Valid`.
- **Edge Cases / Failure Types:** Invalid field types or logic failures are aggregated into an `errors` array, flagging `valid: false`.

##### **Request Is Valid**
- **Type & Technical Role:** `n8n-nodes-base.if` (v2.2) — Branching node evaluating the validation status.
- **Configuration Choices:** Evaluates whether `$json.valid` strictly equals `true`.
- **Key Expressions / Variables:** `={{ $json.valid }}`
- **Input / Output Connections:** Input: `Validate Request` $\rightarrow$ Output (True): `Check Calendar` | Output (False): `Return Invalid`.
- **Edge Cases / Failure Types:** Routes invalid payloads immediately to the error-handling branch without calling external APIs.

##### **Return Invalid**
- **Type & Technical Role:** `n8n-nodes-base.respondToWebhook` (v1.4) — Terminates invalid request executions.
- **Configuration Choices:** Returns HTTP status code `400` with JSON body containing `status: 'invalid_request'` and the collected error messages.
- **Key Expressions / Variables:** `={{ { status: 'invalid_request', errors: $json.errors } }}`
- **Input / Output Connections:** Input: `Request Is Valid` (False branch) $\rightarrow$ Output: None (Terminal response).

---

### Block 1.2: Calendar Availability Check

#### Overview
This block queries the designated Google Calendar to confirm that the requested time interval is completely free, routing busy slots or provider errors to dedicated HTTP response nodes.

#### Nodes Involved
- `Check Calendar`
- `Validate Calendar Result`
- `Interval Is Available`
- `Return Provider Error`
- `Return Busy`

#### Node Details

##### **Check Calendar**
- **Type & Technical Role:** `n8n-nodes-base.googleCalendar` (v1.3) — Queries Google Calendar availability.
- **Configuration Choices:** Resource: `calendar`, Operation: `availability`, Output Format: `availability`. References `YOUR_CALENDAR_ID`.
- **Key Expressions / Variables:** 
  - `timeMin`: `={{ $('Validate Request').first().json.request.start }}`
  - `timeMax`: `={{ $('Validate Request').first().json.request.end }}`
- **Input / Output Connections:** Input: `Request Is Valid` (True branch) $\rightarrow$ Output: `Validate Calendar Result` (Main) or `Return Provider Error` (Error output via `continueErrorOutput`).
- **Version-Specific Requirements:** Requires Google Calendar OAuth2 credentials.
- **Edge Cases / Failure Types:** API timeouts, invalid calendar IDs, or auth token revocation trigger the error output path.

##### **Validate Calendar Result**
- **Type & Technical Role:** `n8n-nodes-base.code` (v2) — Ensures the availability response from Google is structurally valid.
- **Configuration Choices:** Asserts that `typeof $json.available === 'boolean'`.
- **Key Expressions / Variables:** Evaluates `$json.available`.
- **Input / Output Connections:** Input: `Check Calendar` $\rightarrow$ Output: `Interval Is Available` (Main) or `Return Provider Error` (Error output).
- **Edge Cases / Failure Types:** Throws an error if the API response is malformed or missing the boolean flag.

##### **Interval Is Available**
- **Type & Technical Role:** `n8n-nodes-base.if` (v2.2) — Evaluates whether the requested time slot is open.
- **Configuration Choices:** Checks if `$json.available === true`.
- **Key Expressions / Variables:** `={{ $json.available === true }}`
- **Input / Output Connections:** Input: `Validate Calendar Result` $\rightarrow$ Output (True): `Create Appointment` | Output (False): `Return Busy`.

##### **Return Provider Error**
- **Type & Technical Role:** `n8n-nodes-base.respondToWebhook` (v1.4) — Returns a failure response when calendar availability cannot be verified.
- **Configuration Choices:** Sets HTTP response code `502`.
- **Key Expressions / Variables:** Static JSON response indicating unavailability.
- **Input / Output Connections:** Input: `Check Calendar` or `Validate Calendar Result` error outputs $\rightarrow$ Output: None (Terminal response).

##### **Return Busy**
- **Type & Technical Role:** `n8n-nodes-base.respondToWebhook` (v1.4) — Returns a conflict response when the time interval is already occupied.
- **Configuration Choices:** Sets HTTP response code `409`.
- **Key Expressions / Variables:** `={{ { status: 'busy', request_id: $('Validate Request').first().json.request.request_id, message: '...' } }}`
- **Input / Output Connections:** Input: `Interval Is Available` (False branch) $\rightarrow$ Output: None (Terminal response).

---

### Block 1.3: Event Creation & Verification

#### Overview
This block creates the Google Calendar event using the provided `request_id` as the calendar event identifier, performs rigorous post-creation receipt validation, and returns a success response or unconfirmed provider failure.

#### Nodes Involved
- `Create Appointment`
- `Confirm Created Event`
- `Return Booking`
- `Return Unconfirmed`
- `Return Unsatisfied/Unconfirmed` (`Return Unconfirmed` node 13)

#### Node Details

##### **Create Appointment**
- **Type & Technical Role:** `n8n-nodes-base.googleCalendar` (v1.3) — Creates a private event in the target calendar.
- **Configuration Choices:** Resource: `event`, Operation: `create`. Uses `YOUR_CALENDAR_ID`. Configures `visibility` as `private`, `showMeAs` as `opaque`, disables default reminders (`useDefaultReminders: false`), and maps the custom event ID and summary.
- **Key Expressions / Variables:**
  - `start`: `={{ $('Validate Request').first().json.request.start }}`
  - `end`: `={{ $('Validate Request').first().json.request.end }}`
  - `Additional Fields -> id`: `={{ $('Validate Request').first().json.request.request_id }}`
  - `Additional Fields -> summary`: `={{ $('Validate Request').first().json.request.summary }}`
- **Input / Output Connections:** Input: `Interval Is Available` (True branch) $\rightarrow$ Output: `Confirm Created Event` (Main) or `Return Unconfirmed` (Error output via `continueErrorOutput`).
- **Edge Cases / Failure Types:** API failures or collision errors on the custom event ID trigger the error output path.

##### **Confirm Created Event**
- **Type & Technical Role:** `n8n-nodes-base.code` (v2) — Validates the event creation receipt returned by Google Calendar.
- **Configuration Choices:** Checks that `$json.id` matches `request.request_id`, status is `'confirmed'`, and that the returned start and end ISO timestamps strictly match the requested values.
- **Key Expressions / Variables:** Cross-references data from `$('Validate Request')` and `$json`.
- **Input / Output Connections:** Input: `Create Appointment` $\rightarrow$ Output: `Return Booking` (Main) or `Return Unconfirmed` (Error output via `continueErrorOutput`).
- **Edge Cases / Failure Types:** Throws an error if Google returns a mismatched ID, unconfirmed status, or altered time boundaries.

##### **Return Booking**
- **Type & Technical Role:** `n8n-nodes-base.respondToWebhook` (v1.4) — Returns the successful booking confirmation.
- **Configuration Choices:** Sets HTTP response code `201`.
- **Key Expressions / Variables:** `={{ $json }}` (passing the booked appointment summary).
- **Input / Output Connections:** Input: `Confirm Created Event` $\rightarrow$ Output: None (Terminal response).

##### **Return Unconfirmed** (Nodes 13)
- **Type & Technical Role:** `n8n-nodes-base.respondToWebhook` (v1.4) — Handles creation failures or unconfirmed receipts.
- **Configuration Choices:** Sets HTTP response code `502`.
- **Key Expressions / Variables:** `={{ { status: 'unconfirmed', request_id: $('Validate Request').first().json.request.request_id, message: '...' } }}`
- **Input / Output Connections:** Input: Connected to both `Create Appointment` (error output) and `Confirm Created Event` (error output) $\rightarrow$ Output: None (Terminal response).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Setup Instructions | n8n-nodes-base.stickyNote | Documentation and operational guidelines for setup, requests, and limits. | None | None | Book confirmed appointments in Google Calendar... Created by [Benian](https://benian.ai). |
| Receive Request | n8n-nodes-base.webhook | Exposes POST webhook endpoint for booking requests. | External HTTP Caller | Validate Request | Request and validation |
| Validate Request | n8n-nodes-base.code | Parses and validates request body fields against schema and rules. | Receive Request | Request Is Valid | Request and validation |
| Request Is Valid | n8n-nodes-base.if | Routes execution based on request validation status. | Validate Request | Check Calendar, Return Invalid | Request and validation |
| Check Calendar | n8n-nodes-base.googleCalendar | Queries calendar availability for the requested interval. | Request Is Valid | Validate Calendar Result, Return Provider Error | Availability and concurrency |
| Interval Is Available | n8n-nodes-base.if | Branches execution based on calendar availability. | Validate Calendar Result | Create Appointment, Return Busy | Availability and concurrency |
| Create Appointment | n8n-nodes-base.googleCalendar | Creates a private calendar event using the request ID. | Interval Is Available | Confirm Created Event, Return Unconfirmed | Receipt and retries |
| Confirm Created Event | n8n-nodes-base.code | Validates returned event ID, status, and timestamps. | Create Appointment | Return Booking, Return Unconfirmed | Receipt and retries |
| Return Booking | n8n-nodes-base.respondToWebhook | Returns HTTP 201 with confirmed booking details. | Confirm Created Event | None | Receipt and retries |
| Return Invalid | n8n-nodes-base.respondToWebhook | Returns HTTP 400 for invalid request payloads. | Request Is Valid | None | Request and validation |
| Return Provider Error | n8n-nodes-base.respondToWebhook | Returns HTTP 502 when calendar checks fail. | Check Calendar, Validate Calendar Result | None | Availability and concurrency |
| Return Busy | n8n-nodes-base.respondToWebhook | Returns HTTP 409 when the requested slot is busy. | Interval Is Available | None | Availability and concurrency |
| Return Unconfirmed | n8n-nodes-base.respondToWebhook | Returns HTTP 502 when event creation cannot be confirmed. | Create Appointment, Confirm Created Event | None | Receipt and retries |
| Validate Calendar Result | n8n-nodes-base.code | Verifies that availability output is a valid boolean. | Check Calendar | Interval Is Available, Return Provider Error | Availability and concurrency |
| Request and validation notes | n8n-nodes-base.stickyNote | Explains Header Auth, validation rules, and error handling for invalid input. | None | None | Request and validation |
| Availability and concurrency notes | n8n-nodes-base.stickyNote | Details calendar checks, HTTP 409/502 responses, and concurrency considerations. | None | None | Availability and concurrency |
| Receipt and retry notes | n8n-nodes-base.stickyNote | Outlines receipt validation requirements, HTTP 201/502 handling, and retry protocols. | None | None | Receipt and retries |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Webhook Node:**
   - Add a **Webhook** node named `Receive Request`.
   - Set **HTTP Method** to `POST`.
   - Set **Authentication** to `Header Auth`.
   - Set **Path** to `template-calendar-booking`.
   - Set **Response Mode** to `Response Node`.

2. **Add Input Validation Code Node:**
   - Add a **Code** node named `Validate Request`.
   - Connect `Receive Request` (Main) to `Validate Request` (Main).
   - Paste the validation JavaScript snippet from the source workflow into the **JavaScript Code** parameter to parse and validate incoming body parameters (`start`, `end`, `confirmed`, `summary`, `request_id`).

3. **Add Validation Branching (`If` Node):**
   - Add an **If** node named `Request Is Valid`.
   - Connect `Validate Request` (Main) to `Request Is Valid` (Main).
   - Configure condition: Left Value `={{ $json.valid }}`, Operator `Boolean: True`.

4. **Add Invalid Request Response Node:**
   - Add a **Respond to Webhook** node named `Return Invalid`.
   - Connect `Request Is Valid` (False) to `Return Invalid` (Main).
   - Configure **Response Code** to `400`, **Response Body** to `={{ { status: 'invalid_request', errors: $json.errors } }}`.

5. **Add Calendar Availability Check Node:**
   - Add a **Google Calendar** node named `Check Calendar`.
   - Connect `Request Is Valid` (True) to `Check Calendar` (Main).
   - Enable **On Error** $\rightarrow$ `Continue Using Error Output`.
   - Configure Resource to `Calendar`, Operation to `Availability`, and Output Format to `Availability`.
   - Set **Calendar** to `YOUR_CALENDAR_ID` (replace with your calendar ID).
   - Set **Time Min** to `={{ $('Validate Request').first().json.request.start }}` and **Time Max** to `={{ $('Validate Request').first().json.request.end }}`.

6. **Add Calendar Result Validation Code Node:**
   - Add a **Code** node named `Validate Calendar Result`.
   - Connect `Check Calendar` (Main) to `Validate Calendar Result` (Main).
   - Enable **On Error** $\rightarrow$ `Continue Using Error Output`.
   - Connect `Check Calendar` (Error Output) to `Return Provider Error` (Main).
   - Paste the availability verification JavaScript snippet into the node.

7. **Add Provider Error Response Node:**
   - Add a **Respond to Webhook** node named `Return Provider Error`.
   - Connect `Validate Calendar Result` (Error Output) to `Return Provider Error` (Main).
   - Set **Response Code** to `502`.

8. **Add Availability Branching (`If` Node):**
   - Add an **If** node named `Interval Is Available`.
   - Connect `Validate Calendar Result` (Main) to `Interval Is Available` (Main).
   - Configure condition: Left Value `={{ $json.available === true }}`, Operator `Boolean: True`.

9. **Add Busy Response Node:**
   - Add a **Respond to Webhook** node named `Return Busy`.
   - Connect `Interval Is Available` (False) to `Return Busy` (Main).
   - Set **Response Code** to `409`, **Response Body** to return the busy status message with the associated `request_id`.

10. **Add Event Creation Node:**
    - Add a **Google Calendar** node named `Create Appointment`.
    - Connect `Interval Is Available` (True) to `Create Appointment` (Main).
    - Enable **On Error** $\rightarrow$ `Continue Using Error Output`.
    - Configure Resource to `Event`, Operation to `Create`. Set **Calendar** to `YOUR_CALENDAR_ID`.
    - Set **Start** and **End** expressions using `$('Validate Request').first().json.request.start` / `end`.
    - In **Additional Fields**, set **ID** to `={{ $('Validate Request').first().json.request.request_id }}`, **Summary** to `={{ $('Validate Request').first().json.request.summary }}`, **Visibility** to `private`, **Show Me As** to `opaque`, and disable default reminders.

11. **Add Event Receipt Verification Code Node:**
    - Add a **Code** node named `Confirm Created Event`.
    - Connect `Create Appointment` (Main) to `Confirm Created Event` (Main).
    - Enable **On Error** $\rightarrow$ `Continue Using Error Output`.
    - Paste the receipt verification script checking ID match, confirmed status, and timestamps.

12. **Add Unconfirmed Response Node:**
    - Add a **Respond to Webhook** node named `Return Unconfirmed`.
    - Connect error outputs from `Create Appointment` and `Confirm Created Event` to `Return Unconfirmed` (Main).
    - Set **Response Code** to `502`, **Response Body** to return the unconfirmed status JSON.

13. **Add Successful Booking Response Node:**
    - Add a **Respond to Webhook** node named `Return Booking`.
    - Connect `Confirm Created Event` (Main) to `Return Booking` (Main).
    - Set **Response Code** to `201`, **Response Body** to `={{ $json }}`.

14. **Configure Credentials:**
    - Set up and assign a **Webhook Header Auth** credential to `Receive Request`.
    - Set up and assign a **Google Calendar OAuth2** credential to both `Check Calendar` and `Create Appointment`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Created by Benian Technologies | [Benian AI Website](https://benian.ai) |
Enrich LinkedIn profiles with Apollo and display candidate pages in the browser

https://n8nworkflows.xyz/workflows/enrich-linkedin-profiles-with-apollo-and-display-candidate-pages-in-the-browser-13793


# Enrich LinkedIn profiles with Apollo and display candidate pages in the browser

# Workflow Reference: Apollo LinkedIn Profile Viewer

This document provides a detailed technical analysis of the n8n workflow designed to enrich LinkedIn profiles using the Apollo.io API and present the results as a rendered HTML page.

## 1. Workflow Overview

The **Apollo LinkedIn Profile Viewer** workflow automates the process of fetching detailed professional data (email, job history, technology stack) from a LinkedIn URL. It acts as a micro-service: receiving a URL via a webhook, querying Apollo.io, transforming the complex JSON response into a clean format, and returning a styled HTML document.

### Functional Logic Blocks:
- **1.1 Input Reception:** Captures the LinkedIn URL via an HTTP POST request.
- **1.2 Data Enrichment:** Interfaces with the Apollo.io API to match the LinkedIn URL with a professional profile.
- **1.3 Data Normalization:** Uses custom JavaScript to extract and flatten nested API data.
- **1.4 UI Rendering:** Generates a CSS-styled HTML template containing the profile information.
- **1.5 Synchronous Response:** Returns the final HTML directly to the requester.

---

## 2. Block-by-Block Analysis

### 1.1 Webhook Entry Point
**Overview:** This block acts as the trigger. It listens for incoming HTTP POST requests containing the target LinkedIn URL.

*   **Nodes Involved:** `Receive LinkedIn URL via Webhook`
*   **Node Details:**
    *   **Type:** Webhook
    *   **Configuration:** Set to `POST` method. Response mode is set to `lastNode` to ensure the requester receives the generated HTML rather than a simple confirmation.
    *   **Input/Output:** Receives a JSON body (e.g., `{"linkedin_url": "..."}`).
    *   **Edge Cases:** If the request body is missing the `linkedin_url` key, subsequent nodes will fail.

### 1.2 Data Enrichment (Apollo API)
**Overview:** Queries the Apollo.io `/people/match` endpoint to find profile data associated with the provided URL.

*   **Nodes Involved:** `Fetch Profile from Apollo API`
*   **Node Details:**
    *   **Type:** HTTP Request
    *   **Configuration:** Uses `POST` to `https://api.apollo.io/api/v1/people/match`. 
    *   **Authentication:** Requires an `x-api-key` header (currently hardcoded in the node).
    *   **Parameters:** Sends `linkedin_url` as a query parameter.
    *   **Edge Cases:** 
        *   **Auth Error:** Invalid API key will result in a 401/403 error.
        *   **Rate Limits:** Apollo API has strict usage quotas.
        *   **No Match:** If no profile is found, the API returns a success status but empty data.

### 1.3 Data Processing
**Overview:** Cleans the raw, deeply nested JSON response from Apollo into a flat, developer-friendly object.

*   **Nodes Involved:** `Normalize & Extract Profile Fields`
*   **Node Details:**
    *   **Type:** Code (JavaScript)
    *   **Configuration:** "On Error: Continue Regular Output" is enabled to prevent workflow crashes on empty matches.
    *   **Key Logic:** 
        *   Identifies the "current" job from the employment history array.
        *   Maps person and organization attributes (Name, Email, Industry, Tech Stack).
        *   Returns a flat object with standardized keys (e.g., `full_name`, `organization_technologies`).
    *   **Output:** A single JSON object per profile.

### 1.4 HTML Rendering & Response
**Overview:** Converts the structured data into a visual format and sends it back to the user's browser or application.

*   **Nodes Involved:** `Build HTML Profile Cards`, `Return HTML Profile Page`
*   **Node Details:**
    *   **Build HTML Profile Cards (Code):** Uses a template literal to generate a complete HTML document. It includes embedded CSS for card styling and loops through input items to create one card per profile.
    *   **Return HTML Profile Page (Respond to Webhook):** 
        *   **Response Body:** References the `html` string generated in the previous node.
        *   **Response Mode:** Set to `text/html` (implicitly via the Respond to Webhook node configuration) to allow browsers to render the content immediately.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Receive LinkedIn URL via Webhook | Webhook | Trigger / Input | (None) | Fetch Profile from Apollo API | Receives a POST request containing a `linkedin_url` in the body. |
| Fetch Profile from Apollo API | HTTP Request | API Integration | Receive LinkedIn URL via Webhook | Normalize & Extract Profile Fields | Replace the hardcoded `x-api-key` header value with your own Apollo.io API key. |
| Normalize & Extract Profile Fields | Code | Data Transformation | Fetch Profile from Apollo API | Build HTML Profile Cards | Normalizes the raw Apollo API response and maps person and organization fields. |
| Build HTML Profile Cards | Code | UI Generation | Normalize & Extract Profile Fields | Return HTML Profile Page | Generates a styled multi-card HTML page from the normalized profile data. |
| Return HTML Profile Page | Respond to Webhook | Output Delivery | Build HTML Profile Cards | (None) | The output can be opened in any browser. |

---

## 4. Reproducing the Workflow from Scratch

### Prerequisites
- An active **Apollo.io** account and API Key.
- n8n instance (Desktop or Cloud).

### Step-by-Step Setup
1.  **Webhook Trigger:** 
    *   Create a **Webhook** node. 
    *   Set **HTTP Method** to `POST`. 
    *   Set **Path** to `enrich-linkedin`. 
    *   Set **Response Mode** to `When Last Node Finishes`.
2.  **API Call:**
    *   Add an **HTTP Request** node.
    *   **Method:** `POST`.
    *   **URL:** `https://api.apollo.io/api/v1/people/match`.
    *   **Query Parameters:** Add `linkedin_url` with value `{{ $json.body.linkedin_url }}`.
    *   **Headers:** Add `x-api-key` (your Apollo key) and `Content-Type: application/json`.
3.  **Data Extraction:**
    *   Add a **Code** node (JavaScript).
    *   Paste logic to extract `person.name`, `person.email`, and filter the `person.employment_history` array for the item where `current === true`.
    *   Set "On Error" to `Continue` to handle cases where Apollo finds no data.
4.  **HTML Templating:**
    *   Add a **Code** node.
    *   Create a variable `html` using a template literal.
    *   Include a `<style>` block for CSS and a `<div>` structure to display the variables extracted in the previous step.
    *   Ensure the node returns an object like `{ "html": "..." }`.
5.  **Final Response:**
    *   Add a **Respond to Webhook** node.
    *   Set **Respond With** to `Text`.
    *   Set **Response Body** to the output of the HTML node: `{{ $node["Build HTML Profile Cards"].json["html"] }}`.
6.  **Connections:**
    *   Connect the nodes in the linear order described above.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Apollo API Documentation** | [Apollo.io API Reference](https://www.apollo.io/docs/) |
| **Usage Instructions** | Send POST to Webhook URL with body: `{"linkedin_url": "URL"}` |
| **Styling** | Layout is responsive and uses Segoe UI/sans-serif fonts. |
| **Data Privacy** | Ensure your Apollo plan includes the "reveal" permissions required for emails and phone numbers if those fields are needed. |
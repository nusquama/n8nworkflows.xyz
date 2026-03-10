Production AI Playbook: Human Oversight (3 of 3)

https://n8nworkflows.xyz/workflows/production-ai-playbook--human-oversight--3-of-3--13849


# Production AI Playbook: Human Oversight (3 of 3)

### 1. Workflow Overview

This workflow implements a robust **Human-in-the-loop (HITL)** system for AI-generated content. It allows users to submit content via a form, have it automatically enhanced by an AI agent, and then routed to a human reviewer through their preferred channel (Slack or Email). The workflow is designed for production reliability, handling not just "Approve" and "Reject" actions, but also "Timeout" scenarios via an escalation path.

**Logical Blocks:**
- **1.1 Submit & Enhance:** Captures user input and uses an LLM to refine the text.
- **1.2 Route & Review:** Directs the content to Slack or Email and waits for a human response.
- **1.3 Decide & Act:** Processes the reviewer's decision to either publish the content, route it for revision, or escalate to a manager if no response is received.

---

### 2. Block-by-Block Analysis

#### 2.1 Submit & Enhance
**Overview:** This block acts as the entry point, gathering raw content and a choice of communication channel, then using AI to "polish" the text.

*   **On form submission1**
    *   **Type:** Form Trigger
    *   **Configuration:** Displays a form titled "Content Approval Request" with two fields: "Content Body" (Textarea) and "Approval Channel" (Dropdown: slack, email).
    *   **Output:** The raw text and the routing preference.
*   **AI Agent1**
    *   **Type:** AI Agent (LangChain)
    *   **Role:** Refines the input text to be more professional and engaging.
    *   **Key Expression:** `Enhance the following content... Content: {{ $json['Content Body'] }}`.
    *   **Failure Type:** LLM context window limits or API rate limits.
*   **OpenAI Chat Model**
    *   **Type:** Chat OpenAI Model
    *   **Configuration:** Uses `gpt-4o`.
    *   **Role:** Provides the underlying intelligence for the AI Agent.

#### 2.2 Route & Review
**Overview:** A routing logic block that pauses the workflow execution until a human interacts with a notification.

*   **Route by Channel**
    *   **Type:** Switch
    *   **Configuration:** Two outputs based on the "Approval Channel" value from the form (`slack` or `email`).
*   **Slack Approval Request**
    *   **Type:** Slack (Send and Wait)
    *   **Configuration:** Sends a message with "Approve" and "Reject" buttons to a specific user.
    *   **Wait Logic:** Pauses the workflow. Set to timeout after **2 minutes** (for demo purposes).
    *   **Expression:** Includes both the original and AI-enhanced content snippets.
*   **Email Approval Request**
    *   **Type:** Gmail (Send and Wait)
    *   **Configuration:** Sends an interactive email with approval buttons.
    *   **Wait Logic:** Pauses the workflow until the email recipient clicks a choice.

#### 2.3 Decide & Act
**Overview:** This final stage evaluates the outcome of the human interaction and triggers the appropriate business logic.

*   **Slack/Email Decision** (Two separate nodes)
    *   **Type:** Switch
    *   **Logic:** 
        *   **Timed Out:** If the input `data` is empty (no response within the limit).
        *   **Approved:** If `data.approved` is true.
        *   **Rejected:** If `data.approved` is false.
*   **Publish Content**
    *   **Type:** Code Node
    *   **Function:** Simulates the publishing process. Returns a JSON object with a `published` status and timestamp.
*   **Handle Rejection**
    *   **Type:** Code Node
    *   **Function:** Simulates the revision process. Marks status as `rejected` and suggests routing back to the team.
*   **Escalate to Manager**
    *   **Type:** Gmail
    *   **Function:** Sends an escalation email if the reviewer fails to respond within the allotted time.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| On form submission1 | Form Trigger | Input Entry | (None) | AI Agent1 | Submit & Enhance |
| AI Agent1 | AI Agent | Content Refinement | On form submission1 | Route by Channel | Submit & Enhance |
| OpenAI Chat Model | Chat Model | LLM Provider | (None) | AI Agent1 | Submit & Enhance |
| Route by Channel | Switch | Path Routing | AI Agent1 | Slack/Email Approval | Route & Review |
| Slack Approval Request | Slack | Human Interaction | Route by Channel | Slack Decision | Route & Review |
| Email Approval Request | Gmail | Human Interaction | Route by Channel | Email Decision | Route & Review |
| Slack Decision | Switch | Outcome Logic | Slack Approval | Publish/Handle/Escalate | Decide & Act |
| Email Decision | Switch | Outcome Logic | Email Approval | Publish/Handle/Escalate | Decide & Act |
| Publish Content | Code | Success Action | Decision Nodes | (None) | Decide & Act |
| Handle Rejection | Code | Failure Action | Decision Nodes | (None) | Decide & Act |
| Escalate to Manager | Gmail | Timeout Action | Decision Nodes | (None) | Decide & Act |

---

### 4. Reproducing the Workflow from Scratch

1.  **Trigger Setup:**
    *   Create a **Form Trigger** node. Add a `textarea` field ("Content Body") and a `dropdown` field ("Approval Channel") with options `slack` and `email`.
2.  **AI Integration:**
    *   Add an **AI Agent** node. Set the prompt to take the content body and request a professional enhancement.
    *   Attach an **OpenAI Chat Model** node to the AI Agent. Select `gpt-4o` and provide credentials.
3.  **Routing Logic:**
    *   Connect a **Switch** node ("Route by Channel"). Create two rules:
        *   If `Approval Channel` equals `slack` -> Output 1.
        *   If `Approval Channel` equals `email` -> Output 2.
4.  **Wait for Interaction (Slack):**
    *   Create a **Slack** node. Set the operation to **Send and Wait**.
    *   In `Approval Options`, set `Approval Type` to "Double" (Approve/Reject).
    *   Set the message text to show `{{ $json.output }}` (the AI content).
5.  **Wait for Interaction (Email):**
    *   Create a **Gmail** node. Set the operation to **Send and Wait**.
    *   Configure `Approval Options` similarly to the Slack node.
6.  **Branch Handling:**
    *   For both Slack and Email, connect a **Switch** node ("Decision") with 3 rules:
        *   **Timed Out:** String expression `{{ $json.data }}` is Empty.
        *   **Approved:** String expression `{{ $json.data.approved }}` equals `true`.
        *   **Rejected:** String expression `{{ $json.data.approved }}` equals `false`.
7.  **Final Actions:**
    *   Connect the **Approved** outputs to a **Code** node that returns a "status: published" JSON.
    *   Connect the **Rejected** outputs to a **Code** node that returns a "status: rejected" JSON.
    *   Connect the **Timed Out** outputs to a **Gmail** node configured to send an escalation message to a manager's email.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Multi-Channel Content Approval with Slack and Email Escalation | Learning companion to the Production AI Playbook series. |
| Production AI Playbook - Human Oversight Blog | [Link to blog](https://go.n8n.io/PAP-HO-Blog) |
| Timeout Configuration | It is recommended to set timeout durations (e.g., 2-4 hours) for production environments. |
| LLM Requirements | Requires valid OpenAI API credentials for the `gpt-4o` model. |
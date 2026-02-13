Secure and classify legal documents with OpenAI, Airtable and HTML to PDF

https://n8nworkflows.xyz/workflows/secure-and-classify-legal-documents-with-openai--airtable-and-html-to-pdf-12661


# Secure and classify legal documents with OpenAI, Airtable and HTML to PDF

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

# Secure and classify legal documents with OpenAI, Airtable and HTML to PDF — Workflow Reference

## 1. Workflow Overview

This workflow implements an “enterprise legal vault” pattern: it ingests a document, generates a deduplication fingerprint, performs AI-based classification/risk scoring (mocked in the current workflow), fetches policy rules from Airtable, applies tiered PDF security (password locking), and then sets lifecycle retention reminders while syncing downstream systems (HubSpot contact update).

### 1.1 Intake (Trigger)
A single intake node starts the workflow by copying a file from Google Drive (used here as the multi-source intake placeholder).

### 1.2 AI Analysis & Fingerprinting
A Code node adds a SHA-256-like fingerprint and mock AI outputs (jurisdiction, riskScore, docType). In a production variant, this would be replaced by an OpenAI node and a real hashing step.

### 1.3 Policy Rules Retrieval (Airtable)
An Airtable node is intended to fetch or upsert jurisdiction/security rules used for enforcement. In the current configuration, base/table are not set.

### 1.4 Security Enforcement Routing (Risk → Tier)
A Switch routes documents based on `riskScore` into two security paths, each applying PDF locking via HTMLCSSToPDF.

### 1.5 Lifecycle / Retention + Notifications
A Code node calculates expiry and reminder dates, then creates a Google Calendar reminder and updates/creates a HubSpot contact.

---

## 2. Block-by-Block Analysis

## Block 1 — Intake (Trigger)

**Overview:** Starts the workflow by copying a specified Google Drive file. This acts as the entry point and provides the initial JSON payload for downstream nodes.

**Nodes Involved:**
- Trigger: Multi-Source Intake (Google Drive)

### Node: Trigger: Multi-Source Intake
- **Type / Role:** Google Drive node (used as a trigger/entry step; operation: file copy).
- **Configuration (interpreted):**
  - **Operation:** `copy`
  - **File:** A specific file ID pointing to a Google Sheets document (“Copy of Personal Finance Tracker Sheet”).
  - **Options:** none set.
- **Key data produced:** Typically a Drive copy operation returns metadata about the new file (IDs, names, links). Downstream nodes rely on this JSON as `item`.
- **Connections:**
  - **Output →** AI Analysis & Fingerprinting
- **Credentials:** Google Drive OAuth2 (“PDFMunk - Jitesh”).
- **Edge cases / failures:**
  - OAuth token expired / missing scopes (Drive file copy requires appropriate permissions).
  - File ID not accessible to the credential user.
  - Copy operation returns metadata only; if you expect binary content, you must explicitly download the file in an additional step.
- **Version notes:** Node typeVersion 3.

---

## Block 2 — AI Analysis & Fingerprinting

**Overview:** Adds a deduplication fingerprint and AI-derived metadata (jurisdiction, riskScore, docType). This workflow currently uses mock values; it does not call OpenAI despite the title.

**Nodes Involved:**
- AI Analysis & Fingerprinting (Code)

### Node: AI Analysis & Fingerprinting
- **Type / Role:** Code node; enriches the incoming JSON.
- **Configuration (interpreted):**
  - Reads the first incoming item: `$input.first().json`
  - Adds:
    - `fingerprint` as `"SHA256-" + randomString` (not a real SHA-256 hash)
    - `jurisdiction = 'EU (GDPR)'`
    - `riskScore = 95`
    - `docType = 'Trade Secret'`
- **Key expressions / variables:**
  - `$input.first().json`
- **Connections:**
  - **Input ←** Trigger: Multi-Source Intake
  - **Output →** Airtable: Fetch Rules
- **Edge cases / failures:**
  - If upstream returns no items, `$input.first()` will be `undefined` and throw.
  - Risk score is hardcoded to 95, so routing always goes to the highest tier path unless changed.
  - Fingerprinting is non-deterministic; deduplication will not work reliably.
- **Version notes:** Code node typeVersion 2.

---

## Block 3 — Rules/Policy Retrieval (Airtable)

**Overview:** Intended to retrieve and/or persist policy rules (e.g., per jurisdiction) in Airtable. As configured, it uses an Upsert operation but has no base/table selected, so it will fail until configured.

**Nodes Involved:**
- Airtable: Fetch Rules (Airtable)

### Node: Airtable: Fetch Rules
- **Type / Role:** Airtable node; policy rules store (currently configured as upsert).
- **Configuration (interpreted):**
  - **Operation:** `upsert`
  - **Base/Table:** not set (empty values).
  - **Columns mapping:** “defineBelow” mode is selected, but no schema/mapping is defined.
- **Connections:**
  - **Input ←** AI Analysis & Fingerprinting
  - **Output →** Switch: Security Router
- **Credentials:** Airtable Personal Access Token.
- **Edge cases / failures:**
  - Missing base/table causes configuration/runtime error.
  - Upsert requires a unique key / matching columns; none are configured.
  - If the intent is “fetch rules”, likely the operation should be “Search” or “Get Many” rather than Upsert.
- **Version notes:** Airtable node typeVersion 2.

---

## Block 4 — Security Routing & Enforcement (HTML to PDF → PDF Security)

**Overview:** Uses a Switch to choose a security tier based on AI `riskScore`, then applies PDF locking via HTMLCSSToPDF. Two branches exist, but both currently apply the same password lock parameters.

**Nodes Involved:**
- Switch: Security Router (Switch)
- Lock PDF with password (HTMLCSSToPDF)
- Lock PDF with password1 (HTMLCSSToPDF)

### Node: Switch: Security Router
- **Type / Role:** Switch node; risk-based router.
- **Configuration (interpreted):**
  - **Value to evaluate:** `={{ $node["AI Analysis & Fingerprinting"].json.riskScore }}`
  - **Rules (in order):**
    1. `riskScore >= 90` → Output 0
    2. `riskScore >= 50` → Output 1
  - No explicit “else” output is configured; items that match no rules will not continue.
- **Connections:**
  - **Input ←** Airtable: Fetch Rules
  - **Output 0 →** Lock PDF with password
  - **Output 1 →** Lock PDF with password1
- **Edge cases / failures:**
  - If `riskScore` is missing or non-numeric, rule evaluation may misroute or route nowhere.
  - Scores < 50 will be dropped (no default branch).
- **Version notes:** Switch node typeVersion 1.

### Node: Lock PDF with password
- **Type / Role:** HTMLCSSToPDF node; applies PDF security (locking).
- **Configuration (interpreted):**
  - **Resource:** `pdfSecurity`
  - **lock_url:** `user@example.com` (this is suspicious: typically a URL to the PDF to lock is expected; here it’s an email-like string)
  - **lock_password:** `Smartmini@90`
- **Connections:**
  - **Input ←** Switch: Security Router (Output 0)
  - **Output →** Code: Calculate Retention
- **Credentials:** htmlcsstopdfApi (“pdf munk - deepanshi”).
- **Edge cases / failures:**
  - If `lock_url` must be a reachable file URL, this will fail until replaced with an actual file URL/path produced by earlier nodes.
  - Hardcoding passwords is unsafe; should use environment variables or a secret store.
  - External API timeouts/limits.
- **Version notes:** typeVersion 1.

### Node: Lock PDF with password1
- **Type / Role:** Same as above; intended for a different tier, but configured identically.
- **Configuration (interpreted):** Same resource and password; same suspicious `lock_url`.
- **Connections:**
  - **Input ←** Switch: Security Router (Output 1)
  - **Output →** Code: Calculate Retention
- **Credentials / Failures / Notes:** Same as “Lock PDF with password”.
- **Version notes:** typeVersion 1.

---

## Block 5 — Retention/Lifecycle + Downstream Sync

**Overview:** Computes document expiry and a reminder date, creates a Google Calendar reminder, and updates/creates a HubSpot contact.

**Nodes Involved:**
- Code: Calculate Retention (Code)
- Google Calendar: Reminder (Google Calendar)
- Create or update a contact (HubSpot)

### Node: Code: Calculate Retention
- **Type / Role:** Code node; calculates retention metadata.
- **Configuration (interpreted):**
  - Reads `$input.first().json`
  - Sets:
    - `expiryDate` = today + 5 years (ISO string)
    - `reminderDate` = expiryDate - 90 days (ISO string)
  - Returns updated JSON.
- **Connections:**
  - **Input ←** Lock PDF with password OR Lock PDF with password1
  - **Output →** Google Calendar: Reminder
  - **Output →** Create or update a contact
- **Edge cases / failures:**
  - Mutates date objects in-place (`today.setFullYear(...)` then reuses `expiry` and `setDate(...)`), which can be confusing; still produces valid ISO strings but is easy to break if extended to 90/60/30 logic.
  - If upstream returns binary-only data and no JSON, expected fields may be absent (though this node only appends fields).
- **Version notes:** Code node typeVersion 2.

### Node: Google Calendar: Reminder
- **Type / Role:** Google Calendar node; creates a calendar event reminder.
- **Configuration (interpreted):**
  - **Calendar:** “Holidays in India” (calendar id `en.indian#user@example.com`)
  - **Start/End:** both set to `={{ $json.reminderDate }}`
  - **Additional fields:** none.
- **Connections:**
  - **Input ←** Code: Calculate Retention
  - **Output:** none (terminal)
- **Credentials:** Google Calendar OAuth2 (“jitesh.com@gmail.com”).
- **Edge cases / failures:**
  - Start/end being identical can create a zero-duration event; some calendar setups accept it, others may require an end after start.
  - If reminderDate is invalid/missing, API rejects the request.
  - Permissions: calendar might be read-only (e.g., holiday calendars are often not writable).
- **Version notes:** typeVersion 1.

### Node: Create or update a contact
- **Type / Role:** HubSpot node; CRM sync.
- **Configuration (interpreted):**
  - **Operation:** create or update contact (by email)
  - **Email:** hardcoded `user@example.com`
  - **Authentication:** app token
  - **Additional fields:** none.
- **Connections:**
  - **Input ←** Code: Calculate Retention
  - **Output:** none (terminal)
- **Credentials:** HubSpot App Token.
- **Edge cases / failures:**
  - Hardcoded email prevents per-document contact association.
  - HubSpot rate limiting and validation errors.
- **Version notes:** typeVersion 2.2.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Documentation / grouping | — | — | ## ⚖️ Enterprise Legal Vault  / Automated DMS for secure document lifecycle management across GDPR/CCPA. / Core Logic: Security (tiered AES via HTML to PDF Lock), Intelligence (AI Risk Scoring & Jurisdiction Detection), Lifecycle (retention reminders), Routing (HubSpot, Dropbox, SharePoint). / Quick Setup: Airtable rules, store passwords securely, set folder IDs. / Metadata: `DocHash`, `ExpiryDate`, `SecurityTier`. |
| Sticky_Intake_AI | Sticky Note | Documentation for phases 1–2 | — | — | ## 🔍 PHASE 1 & 2: Intake & AI Analysis / Monitors multi-source triggers. Performs SHA-256 hashing for deduplication. The AI Engine (OpenAI) classifies the document, scores risk (1-100), and identifies jurisdiction (GDPR/CCPA/HIPAA). |
| Sticky_Security_Enforcement | Sticky Note | Documentation for phases 3–4 | — | — | ## 🔐 PHASE 3 & 4: Dynamic Security (HTML to PDF) / Translates Airtable Business Rules into encryption. / Tier A: AES-256 + Print/Copy Blocks / Tier B: AES-128 + Dynamic Watermarking / Tier C: Watermark only |
| Sticky_Distribution_Lifecycle | Sticky Note | Documentation for phases 5–6 | — | — | ## 🏁 PHASE 5 & 6: Distribution & Lifecycle / Passed docs sync to HubSpot and Dropbox. Failed docs go to Quarantine. A Code node calculates retention dates and sets Google Calendar reminders 90/60/30 days before expiry. |
| Trigger: Multi-Source Intake | Google Drive | Ingest/copy source document | — | AI Analysis & Fingerprinting | ## 🔍 PHASE 1 & 2: Intake & AI Analysis / Monitors multi-source triggers. Performs SHA-256 hashing for deduplication. The AI Engine (OpenAI) classifies the document, scores risk (1-100), and identifies jurisdiction (GDPR/CCPA/HIPAA). |
| AI Analysis & Fingerprinting | Code | Add fingerprint + AI classification metadata | Trigger: Multi-Source Intake | Airtable: Fetch Rules | ## 🔍 PHASE 1 & 2: Intake & AI Analysis / Monitors multi-source triggers. Performs SHA-256 hashing for deduplication. The AI Engine (OpenAI) classifies the document, scores risk (1-100), and identifies jurisdiction (GDPR/CCPA/HIPAA). |
| Airtable: Fetch Rules | Airtable | Store/fetch business rules (currently upsert placeholder) | AI Analysis & Fingerprinting | Switch: Security Router | ## 🔐 PHASE 3 & 4: Dynamic Security (HTML to PDF) / Translates Airtable Business Rules into encryption. / Tier A: AES-256 + Print/Copy Blocks / Tier B: AES-128 + Dynamic Watermarking / Tier C: Watermark only |
| Switch: Security Router | Switch | Route by riskScore into tier security branches | Airtable: Fetch Rules | Lock PDF with password; Lock PDF with password1 | ## 🔐 PHASE 3 & 4: Dynamic Security (HTML to PDF) / Translates Airtable Business Rules into encryption. / Tier A: AES-256 + Print/Copy Blocks / Tier B: AES-128 + Dynamic Watermarking / Tier C: Watermark only |
| Lock PDF with password | HTMLCSSToPDF | Apply PDF lock (tier branch) | Switch: Security Router | Code: Calculate Retention | ## 🔐 PHASE 3 & 4: Dynamic Security (HTML to PDF) / Translates Airtable Business Rules into encryption. / Tier A: AES-256 + Print/Copy Blocks / Tier B: AES-128 + Dynamic Watermarking / Tier C: Watermark only |
| Lock PDF with password1 | HTMLCSSToPDF | Apply PDF lock (tier branch) | Switch: Security Router | Code: Calculate Retention | ## 🔐 PHASE 3 & 4: Dynamic Security (HTML to PDF) / Translates Airtable Business Rules into encryption. / Tier A: AES-256 + Print/Copy Blocks / Tier B: AES-128 + Dynamic Watermarking / Tier C: Watermark only |
| Code: Calculate Retention | Code | Compute expiry/reminder dates | Lock PDF with password; Lock PDF with password1 | Google Calendar: Reminder; Create or update a contact | ## 🏁 PHASE 5 & 6: Distribution & Lifecycle / Passed docs sync to HubSpot and Dropbox. Failed docs go to Quarantine. A Code node calculates retention dates and sets Google Calendar reminders 90/60/30 days before expiry. |
| Google Calendar: Reminder | Google Calendar | Create retention reminder event | Code: Calculate Retention | — | ## 🏁 PHASE 5 & 6: Distribution & Lifecycle / Passed docs sync to HubSpot and Dropbox. Failed docs go to Quarantine. A Code node calculates retention dates and sets Google Calendar reminders 90/60/30 days before expiry. |
| Create or update a contact | HubSpot | Sync/update CRM contact | Code: Calculate Retention | — | ## 🏁 PHASE 5 & 6: Distribution & Lifecycle / Passed docs sync to HubSpot and Dropbox. Failed docs go to Quarantine. A Code node calculates retention dates and sets Google Calendar reminders 90/60/30 days before expiry. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** in n8n.

2) **Add the intake node**
   - Add node: **Google Drive**
   - Name: `Trigger: Multi-Source Intake`
   - Operation: **Copy**
   - Configure **File**: select or paste the file ID you want to copy.
   - Set **Google Drive OAuth2** credentials (ensure the account can access the source file).

3) **Add AI/fingerprint enrichment**
   - Add node: **Code**
   - Name: `AI Analysis & Fingerprinting`
   - Paste logic that:
     - Reads `$input.first().json`
     - Adds `fingerprint`, `jurisdiction`, `riskScore`, `docType`
   - Connect: `Trigger: Multi-Source Intake` → `AI Analysis & Fingerprinting`
   - (Recommended for real use) Replace the mock with:
     - A real SHA-256 hash (from file bytes or normalized text).
     - An OpenAI node producing `riskScore/jurisdiction/docType` from extracted text.

4) **Add Airtable rules node**
   - Add node: **Airtable**
   - Name: `Airtable: Fetch Rules`
   - Credentials: **Airtable Personal Access Token**
   - Select **Base** and **Table**
   - Decide intent:
     - If you truly want to **fetch rules**, use an operation like **Search** / **Get Many**
     - If you want to **persist analysis results**, keep **Upsert** and configure:
       - Matching/unique column(s) (e.g., `fingerprint` or `DocHash`)
       - Field mapping for `jurisdiction`, `riskScore`, `docType`, etc.
   - Connect: `AI Analysis & Fingerprinting` → `Airtable: Fetch Rules`

5) **Add security routing**
   - Add node: **Switch**
   - Name: `Switch: Security Router`
   - Value 1 expression: `{{$node["AI Analysis & Fingerprinting"].json.riskScore}}`
   - Add rules in order:
     - `>= 90` (Tier A)
     - `>= 50` (Tier B)
   - (Recommended) Add a default/else route for `< 50` (Tier C) to avoid dropping items.
   - Connect: `Airtable: Fetch Rules` → `Switch: Security Router`

6) **Add PDF security enforcement (two branches)**
   - Add node: **HTMLCSSToPDF** (community node `n8n-nodes-htmlcsstopdf`)
   - Name: `Lock PDF with password`
   - Resource: `pdfSecurity`
   - Configure:
     - `lock_url`: provide the actual URL to the PDF to lock (or whatever the node expects as input). Do **not** use an email placeholder.
     - `lock_password`: store securely (n8n credentials, env var, or external secret manager), not hardcoded.
   - Add a second HTMLCSSToPDF node:
     - Name: `Lock PDF with password1`
     - Configure tier-specific parameters (ideally different password/policy per tier).
   - Connect:
     - `Switch: Security Router` output 0 → `Lock PDF with password`
     - `Switch: Security Router` output 1 → `Lock PDF with password1`

7) **Add retention calculation**
   - Add node: **Code**
   - Name: `Code: Calculate Retention`
   - Compute:
     - `expiryDate` (e.g., +5 years)
     - `reminderDate` (e.g., expiry minus 90 days)
   - Connect:
     - `Lock PDF with password` → `Code: Calculate Retention`
     - `Lock PDF with password1` → `Code: Calculate Retention`

8) **Add Google Calendar reminder**
   - Add node: **Google Calendar**
   - Name: `Google Calendar: Reminder`
   - Choose operation to **create an event** (ensure the calendar is writable).
   - Set:
     - Start: `{{$json.reminderDate}}`
     - End: `{{$json.reminderDate}}` (recommended: add duration, e.g., +15 minutes)
   - Configure **Google Calendar OAuth2** credentials.
   - Connect: `Code: Calculate Retention` → `Google Calendar: Reminder`

9) **Add HubSpot sync**
   - Add node: **HubSpot**
   - Name: `Create or update a contact`
   - Operation: create/update contact
   - Auth: **App token**
   - Email: replace hardcoded value with an expression from the document metadata (e.g., `{{$json.ownerEmail}}`) if applicable.
   - Connect: `Code: Calculate Retention` → `Create or update a contact`

10) **(Optional but implied by sticky notes) Add missing distribution/quarantine nodes**
   - Dropbox/SharePoint upload nodes for “passed docs”
   - A quarantine folder move node for “failed docs”
   - Extend retention reminders to 90/60/30 days with additional date calculations and additional calendar events.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Enterprise Legal Vault” concept: tiered encryption, AI risk scoring & jurisdiction detection, retention reminders, routing to HubSpot/Dropbox/SharePoint; store passwords securely; set intake/review/quarantine folder IDs; metadata fields `DocHash`, `ExpiryDate`, `SecurityTier`. | Sticky note: “⚖️ Enterprise Legal Vault” |
| Intake & AI analysis claims SHA-256 deduplication and OpenAI classification; current workflow uses mocked fingerprint and AI outputs in a Code node. | Sticky note: “PHASE 1 & 2: Intake & AI Analysis” |
| Security tiers A/B/C described (AES-256/AES-128/watermark). Current workflow has two identical PDF lock nodes and no watermark-only branch. | Sticky note: “PHASE 3 & 4: Dynamic Security (HTML to PDF)” |
| Distribution & lifecycle mentions HubSpot + Dropbox sync, quarantine handling, and 90/60/30 reminders. Current workflow implements HubSpot update + single 90-day reminder only; Dropbox/quarantine not present. | Sticky note: “PHASE 5 & 6: Distribution & Lifecycle” |
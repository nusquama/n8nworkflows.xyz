Send weekly GA4 WoW email reports with Gemini AI executive summary

https://n8nworkflows.xyz/workflows/send-weekly-ga4-wow-email-reports-with-gemini-ai-executive-summary-13993


# Send weekly GA4 WoW email reports with Gemini AI executive summary

# 1. Workflow Overview

This workflow generates and emails a weekly Google Analytics 4 performance report with week-over-week comparisons and an AI-written executive summary. It is designed for website or app stakeholders who need a concise Monday-morning analytics digest without manually opening GA4.

The workflow is organized into five main logical blocks:

## 1.1 Scheduled Trigger
A weekly schedule starts the workflow every Monday at 8:00 AM. This is the single entry point.

## 1.2 GA4 Data Collection
The workflow fetches 14 GA4 datasets: 7 categories for the current week and the same 7 for the previous week. These categories are overview metrics, top pages, referrals, events, countries, devices, and new-vs-returning users.

## 1.3 Synchronization
A Merge node waits for all 7 current/previous-week query chains to complete before downstream processing begins. This prevents AI and code steps from reading incomplete analytics data.

## 1.4 AI Summary Generation
A Gemini node builds an executive summary from all collected week-over-week analytics values. Its prompt references the GA4 nodes directly by name rather than relying on the Merge payload structure.

## 1.5 Report Rendering and Delivery
A Code node calculates comparative metrics, formats the data into a styled HTML email, optionally embeds the AI summary, and outputs subject/body/recipient fields. An Email node then sends the report via SMTP.

---

# 2. Block-by-Block Analysis

## 2.1 Scheduled Trigger

**Overview:**  
This block starts the workflow automatically once per week. It defines when all downstream GA4 queries should run.

**Nodes Involved:**  
- Weekly Monday Trigger

### Node Details

#### Weekly Monday Trigger
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`  
  Time-based entry node that initiates the workflow on a recurring schedule.
- **Configuration choices:**  
  Configured to run every week on Monday at hour 8.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: none
  - Output: fans out to:
    - GA4 - Overview Current Week
    - GA4 - Top 5 Pages
    - GA4 - Top 5 Referrals
    - GA4 - Top 5 Events
    - GA4 - Top 5 Countries
    - GA4 - Device Breakdown
    - New vs Returning Users Breakdown
- **Version-specific requirements:**  
  Uses node typeVersion `1.3`.
- **Edge cases or potential failure types:**  
  - Workflow timezone may not match intended reporting timezone.
  - Schedule execution depends on workflow being active.
  - If server timezone or workflow timezone is wrong, week boundaries may shift.
- **Sub-workflow reference:**  
  None.

---

## 2.2 GA4 Data Collection

**Overview:**  
This block collects all analytics datasets needed for both the reporting week and the comparison week. Each current-week node triggers its paired previous-week node, creating 7 parallel chains.

**Nodes Involved:**  
- GA4 - Overview Current Week
- GA4 - Overview Previous Week
- GA4 - Top 5 Pages
- GA4 - Top 5 Pages Previous Week
- GA4 - Top 5 Referrals
- GA4 - Top 5 Referrals Previous Week
- GA4 - Top 5 Events
- GA4 - Top 5 Events Previous Week
- GA4 - Top 5 Countries
- GA4 - Top 5 Countries Previous Week
- GA4 - Device Breakdown
- GA4 - Device Breakdown Previous Week
- New vs Returning Users Breakdown
- GA4 - New vs Returning Previous Week

### Node Details

#### GA4 - Overview Current Week
- **Type and technical role:** `n8n-nodes-base.googleAnalytics`  
  Queries high-level GA4 totals for the last completed 7-day period.
- **Configuration choices:**  
  - Date range: custom
  - Start date: 7 days ago, start of day
  - End date: 1 day ago, start of day
  - Property ID: `481410811`
  - Metrics:
    - totalUsers
    - sessions
    - bounceRate via custom expression alias `bounce_rate`
    - averageSessionDuration via alias `average_session_dur`
    - newUsers via alias `new_users`
  - Dimensions: effectively none / generic default dimension payload
  - Execute Once enabled
- **Key expressions or variables used:**  
  - `{{$now.minus({days: 7}).startOf('day').toFormat('yyyy-MM-dd')}}`
  - `{{$now.minus({days: 1}).startOf('day').toFormat('yyyy-MM-dd')}}`
- **Input and output connections:**  
  - Input: Weekly Monday Trigger
  - Output: GA4 - Overview Previous Week
- **Version-specific requirements:**  
  Uses node typeVersion `2`.
- **Edge cases or potential failure types:**  
  - OAuth credential failure
  - Invalid or unauthorized GA4 property ID
  - Metric availability differences across GA4 properties
  - Empty result if property has no data
  - Date boundary issues due to timezone
- **Sub-workflow reference:**  
  None.

#### GA4 - Overview Previous Week
- **Type and technical role:** `n8n-nodes-base.googleAnalytics`  
  Fetches the equivalent overview metrics for the preceding 7-day comparison period.
- **Configuration choices:**  
  - Start date: 14 days ago
  - End date: 8 days ago
  - Same property and metrics as current-week overview
  - Execute Once enabled
- **Key expressions or variables used:**  
  - `{{$now.minus({days: 14}).startOf('day').toFormat('yyyy-MM-dd')}}`
  - `{{$now.minus({days: 8}).startOf('day').toFormat('yyyy-MM-dd')}}`
- **Input and output connections:**  
  - Input: GA4 - Overview Current Week
  - Output: Wait for All GA4 Data (input 0)
- **Version-specific requirements:**  
  TypeVersion `2`.
- **Edge cases or potential failure types:**  
  Same as current-week overview, plus comparison-period gaps.
- **Sub-workflow reference:**  
  None.

#### GA4 - Top 5 Pages
- **Type and technical role:** `n8n-nodes-base.googleAnalytics`  
  Retrieves the top 5 screens/pages for the current week by page/screen views.
- **Configuration choices:**  
  - Limit: 5
  - Metrics:
    - `screenPageViews` aliased as `screen_page_view`
    - `averageSessionDuration` aliased as `average_session_duration`
  - Dimension: `unifiedScreenName`
  - Current-week date window
  - Execute Once enabled
- **Key expressions or variables used:** current-week date expressions.
- **Input and output connections:**  
  - Input: Weekly Monday Trigger
  - Output: GA4 - Top 5 Pages Previous Week
- **Version-specific requirements:**  
  TypeVersion `2`.
- **Edge cases or potential failure types:**  
  - Empty `unifiedScreenName`
  - Low-traffic properties returning fewer than 5 rows
  - Page/screen naming inconsistency between weeks
- **Sub-workflow reference:**  
  None.

#### GA4 - Top 5 Pages Previous Week
- **Type and technical role:** `n8n-nodes-base.googleAnalytics`  
  Retrieves the previous week’s top 5 pages for WoW page comparison.
- **Configuration choices:**  
  Same as current-week pages, but previous-week dates.
- **Key expressions or variables used:** previous-week date expressions.
- **Input and output connections:**  
  - Input: GA4 - Top 5 Pages
  - Output: Wait for All GA4 Data (input 1)
- **Version-specific requirements:**  
  TypeVersion `2`.
- **Edge cases or potential failure types:**  
  Same as current-week pages.
- **Sub-workflow reference:**  
  None.

#### GA4 - Top 5 Referrals
- **Type and technical role:** `n8n-nodes-base.googleAnalytics`  
  Fetches top traffic sources/mediums for the current week.
- **Configuration choices:**  
  - Limit: 5
  - Metric: sessions
  - Dimension: `sessionSourceMedium`
  - Ordered descending by sessions
  - Current-week dates
  - Execute Once enabled
- **Key expressions or variables used:** current-week date expressions.
- **Input and output connections:**  
  - Input: Weekly Monday Trigger
  - Output: GA4 - Top 5 Referrals Previous Week
- **Version-specific requirements:**  
  TypeVersion `2`.
- **Edge cases or potential failure types:**  
  - Direct traffic represented as `(direct) / (none)`
  - Fewer than 5 rows
  - Unusual source formatting from app analytics
- **Sub-workflow reference:**  
  None.

#### GA4 - Top 5 Referrals Previous Week
- **Type and technical role:** `n8n-nodes-base.googleAnalytics`  
  Fetches previous-week top traffic sources for comparison.
- **Configuration choices:**  
  Same as current-week referrals, but previous-week dates.
- **Key expressions or variables used:** previous-week date expressions.
- **Input and output connections:**  
  - Input: GA4 - Top 5 Referrals
  - Output: Wait for All GA4 Data (input 2)
- **Version-specific requirements:**  
  TypeVersion `2`.
- **Edge cases or potential failure types:**  
  Same as current-week referrals.
- **Sub-workflow reference:**  
  None.

#### GA4 - Top 5 Events
- **Type and technical role:** `n8n-nodes-base.googleAnalytics`  
  Pulls top events for the current week by event count.
- **Configuration choices:**  
  - Limit: 5
  - Metric: `eventCount` aliased as `event_count`
  - Dimension: `eventName`
  - Ordered descending by event count
  - Current-week dates
  - Execute Once enabled
- **Key expressions or variables used:** current-week date expressions.
- **Input and output connections:**  
  - Input: Weekly Monday Trigger
  - Output: GA4 - Top 5 Events Previous Week
- **Version-specific requirements:**  
  TypeVersion `2`.
- **Edge cases or potential failure types:**  
  - Ordering uses metric name `=event_count`, which may be brittle depending on node parsing/version.
  - Top events can be dominated by technical/system events.
  - Fewer than 5 rows returned.
- **Sub-workflow reference:**  
  None.

#### GA4 - Top 5 Events Previous Week
- **Type and technical role:** `n8n-nodes-base.googleAnalytics`  
  Retrieves previous-week top event data for WoW event comparison.
- **Configuration choices:**  
  Same as current-week events, with previous-week dates.
- **Key expressions or variables used:** previous-week date expressions.
- **Input and output connections:**  
  - Input: GA4 - Top 5 Events
  - Output: Wait for All GA4 Data (input 3)
- **Version-specific requirements:**  
  TypeVersion `2`.
- **Edge cases or potential failure types:**  
  Same as current-week events.
- **Sub-workflow reference:**  
  None.

#### GA4 - Top 5 Countries
- **Type and technical role:** `n8n-nodes-base.googleAnalytics`  
  Gets top 5 countries by sessions for the current week.
- **Configuration choices:**  
  - Limit: 5
  - Metric: sessions
  - Dimension: country
  - Ordered descending by sessions
  - Current-week dates
  - Execute Once enabled
- **Key expressions or variables used:** current-week date expressions.
- **Input and output connections:**  
  - Input: Weekly Monday Trigger
  - Output: GA4 - Top 5 Countries Previous Week
- **Version-specific requirements:**  
  TypeVersion `2`.
- **Edge cases or potential failure types:**  
  - Country names may vary or be unavailable if analytics data quality is low.
  - Fewer than 5 rows.
- **Sub-workflow reference:**  
  None.

#### GA4 - Top 5 Countries Previous Week
- **Type and technical role:** `n8n-nodes-base.googleAnalytics`  
  Gets previous-week top country data.
- **Configuration choices:**  
  Same as current-week countries, with previous-week dates.
- **Key expressions or variables used:** previous-week date expressions.
- **Input and output connections:**  
  - Input: GA4 - Top 5 Countries
  - Output: Wait for All GA4 Data (input 4)
- **Version-specific requirements:**  
  TypeVersion `2`.
- **Edge cases or potential failure types:**  
  Same as current-week countries.
- **Sub-workflow reference:**  
  None.

#### GA4 - Device Breakdown
- **Type and technical role:** `n8n-nodes-base.googleAnalytics`  
  Retrieves current-week sessions grouped by device category.
- **Configuration choices:**  
  - Metric: sessions
  - Dimension: deviceCategory
  - Ordered descending by sessions
  - Current-week dates
  - Execute Once enabled
- **Key expressions or variables used:** current-week date expressions.
- **Input and output connections:**  
  - Input: Weekly Monday Trigger
  - Output: GA4 - Device Breakdown Previous Week
- **Version-specific requirements:**  
  TypeVersion `2`.
- **Edge cases or potential failure types:**  
  - Categories may include unexpected values beyond mobile/desktop/tablet.
  - Fewer categories than expected.
- **Sub-workflow reference:**  
  None.

#### GA4 - Device Breakdown Previous Week
- **Type and technical role:** `n8n-nodes-base.googleAnalytics`  
  Retrieves previous-week device category data.
- **Configuration choices:**  
  Same as current-week devices, but previous-week dates.
- **Key expressions or variables used:** previous-week date expressions.
- **Input and output connections:**  
  - Input: GA4 - Device Breakdown
  - Output: Wait for All GA4 Data (input 5)
- **Version-specific requirements:**  
  TypeVersion `2`.
- **Edge cases or potential failure types:**  
  Same as current-week devices.
- **Sub-workflow reference:**  
  None.

#### New vs Returning Users Breakdown
- **Type and technical role:** `n8n-nodes-base.googleAnalytics`  
  Retrieves current-week audience loyalty segmentation.
- **Configuration choices:**  
  - Metrics include sessions and default total users metric
  - Dimension: `newVsReturning`
  - Current-week dates
  - Execute Once enabled
- **Key expressions or variables used:** current-week date expressions.
- **Input and output connections:**  
  - Input: Weekly Monday Trigger
  - Output: GA4 - New vs Returning Previous Week
- **Version-specific requirements:**  
  TypeVersion `2`.
- **Edge cases or potential failure types:**  
  - Segment names may not exactly match expected lowercase values in downstream logic.
  - Empty or sparse audience data.
- **Sub-workflow reference:**  
  None.

#### GA4 - New vs Returning Previous Week
- **Type and technical role:** `n8n-nodes-base.googleAnalytics`  
  Retrieves previous-week audience loyalty segmentation.
- **Configuration choices:**  
  Same as current-week new-vs-returning node, but previous-week dates.
- **Key expressions or variables used:** previous-week date expressions.
- **Input and output connections:**  
  - Input: New vs Returning Users Breakdown
  - Output: Wait for All GA4 Data (input 6)
- **Version-specific requirements:**  
  TypeVersion `2`.
- **Edge cases or potential failure types:**  
  Same as current-week node.
- **Sub-workflow reference:**  
  None.

---

## 2.3 Synchronization

**Overview:**  
This block ensures all seven GA4 query chains have completed before AI generation begins. It acts as a barrier to avoid partial execution.

**Nodes Involved:**  
- Wait for All GA4 Data

### Node Details

#### Wait for All GA4 Data
- **Type and technical role:** `n8n-nodes-base.merge`  
  Combines multiple input streams and waits until all configured inputs are available.
- **Configuration choices:**  
  - Mode: `combine`
  - Combine by: `combineByPosition`
  - Number of inputs: `7`
  - Include unpaired: `true`
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Inputs:
    - GA4 - Overview Previous Week
    - GA4 - Top 5 Pages Previous Week
    - GA4 - Top 5 Referrals Previous Week
    - GA4 - Top 5 Events Previous Week
    - GA4 - Top 5 Countries Previous Week
    - GA4 - Device Breakdown Previous Week
    - GA4 - New vs Returning Previous Week
  - Output:
    - Generate AI Summary
- **Version-specific requirements:**  
  Uses typeVersion `3.2`.
- **Edge cases or potential failure types:**  
  - If one upstream GA4 branch fails, downstream execution stops.
  - `includeUnpaired: true` can pass incomplete combinations in some merge scenarios, but here the main purpose is synchronization.
  - Since downstream nodes reference upstream data by node name, the exact merged payload is less important than completion timing.
- **Sub-workflow reference:**  
  None.

---

## 2.4 AI Summary Generation

**Overview:**  
This block asks Gemini to convert raw GA4 comparisons into a short executive summary. The prompt is heavily structured and includes explicit formatting rules for the output.

**Nodes Involved:**  
- Generate AI Summary

### Node Details

#### Generate AI Summary
- **Type and technical role:** `@n8n/n8n-nodes-langchain.googleGemini`  
  Calls Google Gemini to generate a natural-language summary from workflow data.
- **Configuration choices:**  
  - Model: `models/gemini-3.1-flash-lite-preview`
  - A single prompt message is constructed using direct references to all current and previous GA4 nodes.
  - Required output constraints:
    - exactly 3 to 5 bullets
    - each bullet starts with `-`
    - one sentence per bullet
    - max 25 words
    - no markdown headers/bold
- **Key expressions or variables used:**  
  Extensive node references such as:
  - `$('GA4 - Overview Current Week').first().json.totalUsers`
  - `$('GA4 - Top 5 Pages').all().map(...)`
  - `$('GA4 - Top 5 Pages Previous Week').all().map(...)`
  - similar expressions for referrals, events, countries, devices, and new-vs-returning
- **Input and output connections:**  
  - Input: Wait for All GA4 Data
  - Output: Build Report & Email HTML
- **Version-specific requirements:**  
  Uses typeVersion `1.1`; requires the LangChain Gemini node package to be available in the n8n instance.
- **Edge cases or potential failure types:**  
  - Missing Gemini credentials
  - Model availability changes or preview model retirement
  - Prompt may exceed limits if datasets grow, though currently constrained
  - Output format may not fully respect rules
  - Empty or malformed AI response
- **Sub-workflow reference:**  
  None.

---

## 2.5 Report Rendering and Email Delivery

**Overview:**  
This block transforms raw GA4 and Gemini outputs into a styled HTML email and sends it to recipients. It contains almost all formatting, fallback logic, comparison math, and final delivery settings.

**Nodes Involved:**  
- Build Report & Email HTML
- Send Weekly Report

### Node Details

#### Build Report & Email HTML
- **Type and technical role:** `n8n-nodes-base.code`  
  JavaScript processing node that aggregates analytics data, computes WoW deltas, creates HTML, and emits email fields.
- **Configuration choices:**  
  Major functions performed:
  1. Reads all relevant GA4 node outputs by node name
  2. Normalizes values
     - empty page names become `(empty)`
     - direct source `(direct) / (none)` becomes `Direct / App Open`
  3. Excludes technical events:
     - `first_visit`
     - `first_open`
     - `page_view`
     - `app_remove`
     - `os_update`
     - `app_update`
     - `app_store_refund`
  4. Parses Gemini output from several possible response shapes
  5. Hides AI section if Gemini text is empty
  6. Calculates WoW percentage changes, with special favorable logic for bounce rate
  7. Builds a branded HTML email with sections:
     - header KPI strip
     - AI Executive Summary
     - Overview
     - Audience
     - Top Screens
     - Traffic Sources
     - Top Events
     - Geography
     - Devices
  8. Outputs:
     - `subject`
     - `html`
     - `recipients`
     - `_debug`
- **Key expressions or variables used:**  
  Direct node references, for example:
  - `$('GA4 - Overview Current Week').first().json`
  - `$('Generate AI Summary').first().json`
  - `$now.minus({days: 7})...`
  Helper functions include:
  - `wowChange`
  - `wowChangeBounce`
  - `listWoWWithPrev`
  - `formatDuration`
  - `fmt`
  - `fmtPct`
- **Input and output connections:**  
  - Input: Generate AI Summary
  - Output: Send Weekly Report
- **Version-specific requirements:**  
  Uses Code node typeVersion `2`.
- **Edge cases or potential failure types:**  
  - If upstream node names are changed, all `$('<node name>')` references break.
  - If GA4 fields differ from expected keys, rendering or calculations may fail.
  - `newVsReturning` logic expects values like `new` and `returning`.
  - HTML may render differently across email clients.
  - The `fromEmail` used later may not match allowed SMTP sender configuration.
  - AI parsing is defensive but still depends on Gemini returning a supported structure.
- **Sub-workflow reference:**  
  None.

#### Send Weekly Report
- **Type and technical role:** `n8n-nodes-base.emailSend`  
  Sends the final HTML email using SMTP credentials.
- **Configuration choices:**  
  - HTML body: `{{$json.html}}`
  - Subject: `{{$json.subject}}`
  - To: `{{$json.recipients}}, jignesh.sanghani@dev.appstonelab.com`
  - From: `{{$json.recipients}}`
- **Key expressions or variables used:**  
  - `{{ $json.html }}`
  - `{{ $json.subject }}`
  - `{{ $json.recipients }}`
- **Input and output connections:**  
  - Input: Build Report & Email HTML
  - Output: none
- **Version-specific requirements:**  
  TypeVersion `2.1`.
- **Edge cases or potential failure types:**  
  - SMTP auth failure
  - Invalid sender address because `fromEmail` is dynamically set to the recipient value
  - Multiple recipients formatting may fail depending on SMTP server rules
  - HTML body size or spam filtering issues
- **Sub-workflow reference:**  
  None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Weekly Monday Trigger | n8n-nodes-base.scheduleTrigger | Weekly entry point for the report workflow |  | GA4 - Overview Current Week; GA4 - Top 5 Pages; GA4 - Top 5 Referrals; GA4 - Top 5 Events; GA4 - Top 5 Countries; GA4 - Device Breakdown; New vs Returning Users Breakdown | ## ⏰ Schedule Trigger<br>Fires every **Monday at 8:00 AM IST**.<br>To change timezone or time:<br>Open workflow settings → change the Timezone to your preferred one.<br>Default timezone: Your Local Timezone |
| GA4 - Overview Current Week | n8n-nodes-base.googleAnalytics | Current-week overview KPI query | Weekly Monday Trigger | GA4 - Overview Previous Week | ## 📡 Google Analytics 4 — Data Nodes<br>Fetches data for 7 key categories, running both **Current Week** and **Previous Week** nodes to generate deep Week-over-Week (WoW) comparisons.<br>**⚠️ Required: Set your Property ID**<br>Open each of the 14 nodes → Property ID field → replace {YOUR_PROPERTY_ID} with your GA4 numeric property ID.<br>Found at: GA4 Admin → Property Settings → Property ID<br>**Node breakdown (Each has a Current & Previous week version):**<br>- **Overview** — totals for sessions, users, bounce rate, etc.<br>- **Top 5 Pages** — screens/pages by views<br>- **Top 5 Referrals** — traffic sources by sessions<br>- **Top 5 Events** — user interactions by count<br>- **Top 5 Countries** — geographic breakdown by sessions<br>- **Device Breakdown** — mobile / desktop / tablet split<br>- **New vs Returning** — audience loyalty breakdown<br>**All nodes have Execute Once = ON** to prevent duplicate rows. |
| GA4 - Overview Previous Week | n8n-nodes-base.googleAnalytics | Previous-week overview KPI query | GA4 - Overview Current Week | Wait for All GA4 Data | ## 📡 Google Analytics 4 — Data Nodes<br>Fetches data for 7 key categories, running both **Current Week** and **Previous Week** nodes to generate deep Week-over-Week (WoW) comparisons.<br>**⚠️ Required: Set your Property ID**<br>Open each of the 14 nodes → Property ID field → replace {YOUR_PROPERTY_ID} with your GA4 numeric property ID.<br>Found at: GA4 Admin → Property Settings → Property ID<br>**Node breakdown (Each has a Current & Previous week version):**<br>- **Overview** — totals for sessions, users, bounce rate, etc.<br>- **Top 5 Pages** — screens/pages by views<br>- **Top 5 Referrals** — traffic sources by sessions<br>- **Top 5 Events** — user interactions by count<br>- **Top 5 Countries** — geographic breakdown by sessions<br>- **Device Breakdown** — mobile / desktop / tablet split<br>- **New vs Returning** — audience loyalty breakdown<br>**All nodes have Execute Once = ON** to prevent duplicate rows. |
| GA4 - Top 5 Pages | n8n-nodes-base.googleAnalytics | Current-week top pages/screens query | Weekly Monday Trigger | GA4 - Top 5 Pages Previous Week | ## 📡 Google Analytics 4 — Data Nodes<br>Fetches data for 7 key categories, running both **Current Week** and **Previous Week** nodes to generate deep Week-over-Week (WoW) comparisons.<br>**⚠️ Required: Set your Property ID**<br>Open each of the 14 nodes → Property ID field → replace {YOUR_PROPERTY_ID} with your GA4 numeric property ID.<br>Found at: GA4 Admin → Property Settings → Property ID<br>**Node breakdown (Each has a Current & Previous week version):**<br>- **Overview** — totals for sessions, users, bounce rate, etc.<br>- **Top 5 Pages** — screens/pages by views<br>- **Top 5 Referrals** — traffic sources by sessions<br>- **Top 5 Events** — user interactions by count<br>- **Top 5 Countries** — geographic breakdown by sessions<br>- **Device Breakdown** — mobile / desktop / tablet split<br>- **New vs Returning** — audience loyalty breakdown<br>**All nodes have Execute Once = ON** to prevent duplicate rows. |
| GA4 - Top 5 Pages Previous Week | n8n-nodes-base.googleAnalytics | Previous-week top pages/screens query | GA4 - Top 5 Pages | Wait for All GA4 Data | ## 📡 Google Analytics 4 — Data Nodes<br>Fetches data for 7 key categories, running both **Current Week** and **Previous Week** nodes to generate deep Week-over-Week (WoW) comparisons.<br>**⚠️ Required: Set your Property ID**<br>Open each of the 14 nodes → Property ID field → replace {YOUR_PROPERTY_ID} with your GA4 numeric property ID.<br>Found at: GA4 Admin → Property Settings → Property ID<br>**Node breakdown (Each has a Current & Previous week version):**<br>- **Overview** — totals for sessions, users, bounce rate, etc.<br>- **Top 5 Pages** — screens/pages by views<br>- **Top 5 Referrals** — traffic sources by sessions<br>- **Top 5 Events** — user interactions by count<br>- **Top 5 Countries** — geographic breakdown by sessions<br>- **Device Breakdown** — mobile / desktop / tablet split<br>- **New vs Returning** — audience loyalty breakdown<br>**All nodes have Execute Once = ON** to prevent duplicate rows. |
| GA4 - Top 5 Referrals | n8n-nodes-base.googleAnalytics | Current-week top source/medium query | Weekly Monday Trigger | GA4 - Top 5 Referrals Previous Week | ## 📡 Google Analytics 4 — Data Nodes<br>Fetches data for 7 key categories, running both **Current Week** and **Previous Week** nodes to generate deep Week-over-Week (WoW) comparisons.<br>**⚠️ Required: Set your Property ID**<br>Open each of the 14 nodes → Property ID field → replace {YOUR_PROPERTY_ID} with your GA4 numeric property ID.<br>Found at: GA4 Admin → Property Settings → Property ID<br>**Node breakdown (Each has a Current & Previous week version):**<br>- **Overview** — totals for sessions, users, bounce rate, etc.<br>- **Top 5 Pages** — screens/pages by views<br>- **Top 5 Referrals** — traffic sources by sessions<br>- **Top 5 Events** — user interactions by count<br>- **Top 5 Countries** — geographic breakdown by sessions<br>- **Device Breakdown** — mobile / desktop / tablet split<br>- **New vs Returning** — audience loyalty breakdown<br>**All nodes have Execute Once = ON** to prevent duplicate rows. |
| GA4 - Top 5 Referrals Previous Week | n8n-nodes-base.googleAnalytics | Previous-week top source/medium query | GA4 - Top 5 Referrals | Wait for All GA4 Data | ## 📡 Google Analytics 4 — Data Nodes<br>Fetches data for 7 key categories, running both **Current Week** and **Previous Week** nodes to generate deep Week-over-Week (WoW) comparisons.<br>**⚠️ Required: Set your Property ID**<br>Open each of the 14 nodes → Property ID field → replace {YOUR_PROPERTY_ID} with your GA4 numeric property ID.<br>Found at: GA4 Admin → Property Settings → Property ID<br>**Node breakdown (Each has a Current & Previous week version):**<br>- **Overview** — totals for sessions, users, bounce rate, etc.<br>- **Top 5 Pages** — screens/pages by views<br>- **Top 5 Referrals** — traffic sources by sessions<br>- **Top 5 Events** — user interactions by count<br>- **Top 5 Countries** — geographic breakdown by sessions<br>- **Device Breakdown** — mobile / desktop / tablet split<br>- **New vs Returning** — audience loyalty breakdown<br>**All nodes have Execute Once = ON** to prevent duplicate rows. |
| GA4 - Top 5 Events | n8n-nodes-base.googleAnalytics | Current-week top events query | Weekly Monday Trigger | GA4 - Top 5 Events Previous Week | ## 📡 Google Analytics 4 — Data Nodes<br>Fetches data for 7 key categories, running both **Current Week** and **Previous Week** nodes to generate deep Week-over-Week (WoW) comparisons.<br>**⚠️ Required: Set your Property ID**<br>Open each of the 14 nodes → Property ID field → replace {YOUR_PROPERTY_ID} with your GA4 numeric property ID.<br>Found at: GA4 Admin → Property Settings → Property ID<br>**Node breakdown (Each has a Current & Previous week version):**<br>- **Overview** — totals for sessions, users, bounce rate, etc.<br>- **Top 5 Pages** — screens/pages by views<br>- **Top 5 Referrals** — traffic sources by sessions<br>- **Top 5 Events** — user interactions by count<br>- **Top 5 Countries** — geographic breakdown by sessions<br>- **Device Breakdown** — mobile / desktop / tablet split<br>- **New vs Returning** — audience loyalty breakdown<br>**All nodes have Execute Once = ON** to prevent duplicate rows. |
| GA4 - Top 5 Events Previous Week | n8n-nodes-base.googleAnalytics | Previous-week top events query | GA4 - Top 5 Events | Wait for All GA4 Data | ## 📡 Google Analytics 4 — Data Nodes<br>Fetches data for 7 key categories, running both **Current Week** and **Previous Week** nodes to generate deep Week-over-Week (WoW) comparisons.<br>**⚠️ Required: Set your Property ID**<br>Open each of the 14 nodes → Property ID field → replace {YOUR_PROPERTY_ID} with your GA4 numeric property ID.<br>Found at: GA4 Admin → Property Settings → Property ID<br>**Node breakdown (Each has a Current & Previous week version):**<br>- **Overview** — totals for sessions, users, bounce rate, etc.<br>- **Top 5 Pages** — screens/pages by views<br>- **Top 5 Referrals** — traffic sources by sessions<br>- **Top 5 Events** — user interactions by count<br>- **Top 5 Countries** — geographic breakdown by sessions<br>- **Device Breakdown** — mobile / desktop / tablet split<br>- **New vs Returning** — audience loyalty breakdown<br>**All nodes have Execute Once = ON** to prevent duplicate rows. |
| GA4 - Top 5 Countries | n8n-nodes-base.googleAnalytics | Current-week top countries query | Weekly Monday Trigger | GA4 - Top 5 Countries Previous Week | ## 📡 Google Analytics 4 — Data Nodes<br>Fetches data for 7 key categories, running both **Current Week** and **Previous Week** nodes to generate deep Week-over-Week (WoW) comparisons.<br>**⚠️ Required: Set your Property ID**<br>Open each of the 14 nodes → Property ID field → replace {YOUR_PROPERTY_ID} with your GA4 numeric property ID.<br>Found at: GA4 Admin → Property Settings → Property ID<br>**Node breakdown (Each has a Current & Previous week version):**<br>- **Overview** — totals for sessions, users, bounce rate, etc.<br>- **Top 5 Pages** — screens/pages by views<br>- **Top 5 Referrals** — traffic sources by sessions<br>- **Top 5 Events** — user interactions by count<br>- **Top 5 Countries** — geographic breakdown by sessions<br>- **Device Breakdown** — mobile / desktop / tablet split<br>- **New vs Returning** — audience loyalty breakdown<br>**All nodes have Execute Once = ON** to prevent duplicate rows. |
| GA4 - Top 5 Countries Previous Week | n8n-nodes-base.googleAnalytics | Previous-week top countries query | GA4 - Top 5 Countries | Wait for All GA4 Data | ## 📡 Google Analytics 4 — Data Nodes<br>Fetches data for 7 key categories, running both **Current Week** and **Previous Week** nodes to generate deep Week-over-Week (WoW) comparisons.<br>**⚠️ Required: Set your Property ID**<br>Open each of the 14 nodes → Property ID field → replace {YOUR_PROPERTY_ID} with your GA4 numeric property ID.<br>Found at: GA4 Admin → Property Settings → Property ID<br>**Node breakdown (Each has a Current & Previous week version):**<br>- **Overview** — totals for sessions, users, bounce rate, etc.<br>- **Top 5 Pages** — screens/pages by views<br>- **Top 5 Referrals** — traffic sources by sessions<br>- **Top 5 Events** — user interactions by count<br>- **Top 5 Countries** — geographic breakdown by sessions<br>- **Device Breakdown** — mobile / desktop / tablet split<br>- **New vs Returning** — audience loyalty breakdown<br>**All nodes have Execute Once = ON** to prevent duplicate rows. |
| GA4 - Device Breakdown | n8n-nodes-base.googleAnalytics | Current-week device-category query | Weekly Monday Trigger | GA4 - Device Breakdown Previous Week | ## 📡 Google Analytics 4 — Data Nodes<br>Fetches data for 7 key categories, running both **Current Week** and **Previous Week** nodes to generate deep Week-over-Week (WoW) comparisons.<br>**⚠️ Required: Set your Property ID**<br>Open each of the 14 nodes → Property ID field → replace {YOUR_PROPERTY_ID} with your GA4 numeric property ID.<br>Found at: GA4 Admin → Property Settings → Property ID<br>**Node breakdown (Each has a Current & Previous week version):**<br>- **Overview** — totals for sessions, users, bounce rate, etc.<br>- **Top 5 Pages** — screens/pages by views<br>- **Top 5 Referrals** — traffic sources by sessions<br>- **Top 5 Events** — user interactions by count<br>- **Top 5 Countries** — geographic breakdown by sessions<br>- **Device Breakdown** — mobile / desktop / tablet split<br>- **New vs Returning** — audience loyalty breakdown<br>**All nodes have Execute Once = ON** to prevent duplicate rows. |
| GA4 - Device Breakdown Previous Week | n8n-nodes-base.googleAnalytics | Previous-week device-category query | GA4 - Device Breakdown | Wait for All GA4 Data | ## 📡 Google Analytics 4 — Data Nodes<br>Fetches data for 7 key categories, running both **Current Week** and **Previous Week** nodes to generate deep Week-over-Week (WoW) comparisons.<br>**⚠️ Required: Set your Property ID**<br>Open each of the 14 nodes → Property ID field → replace {YOUR_PROPERTY_ID} with your GA4 numeric property ID.<br>Found at: GA4 Admin → Property Settings → Property ID<br>**Node breakdown (Each has a Current & Previous week version):**<br>- **Overview** — totals for sessions, users, bounce rate, etc.<br>- **Top 5 Pages** — screens/pages by views<br>- **Top 5 Referrals** — traffic sources by sessions<br>- **Top 5 Events** — user interactions by count<br>- **Top 5 Countries** — geographic breakdown by sessions<br>- **Device Breakdown** — mobile / desktop / tablet split<br>- **New vs Returning** — audience loyalty breakdown<br>**All nodes have Execute Once = ON** to prevent duplicate rows. |
| New vs Returning Users Breakdown | n8n-nodes-base.googleAnalytics | Current-week new-vs-returning audience query | Weekly Monday Trigger | GA4 - New vs Returning Previous Week | ## 📡 Google Analytics 4 — Data Nodes<br>Fetches data for 7 key categories, running both **Current Week** and **Previous Week** nodes to generate deep Week-over-Week (WoW) comparisons.<br>**⚠️ Required: Set your Property ID**<br>Open each of the 14 nodes → Property ID field → replace {YOUR_PROPERTY_ID} with your GA4 numeric property ID.<br>Found at: GA4 Admin → Property Settings → Property ID<br>**Node breakdown (Each has a Current & Previous week version):**<br>- **Overview** — totals for sessions, users, bounce rate, etc.<br>- **Top 5 Pages** — screens/pages by views<br>- **Top 5 Referrals** — traffic sources by sessions<br>- **Top 5 Events** — user interactions by count<br>- **Top 5 Countries** — geographic breakdown by sessions<br>- **Device Breakdown** — mobile / desktop / tablet split<br>- **New vs Returning** — audience loyalty breakdown<br>**All nodes have Execute Once = ON** to prevent duplicate rows. |
| GA4 - New vs Returning Previous Week | n8n-nodes-base.googleAnalytics | Previous-week new-vs-returning audience query | New vs Returning Users Breakdown | Wait for All GA4 Data | ## 📡 Google Analytics 4 — Data Nodes<br>Fetches data for 7 key categories, running both **Current Week** and **Previous Week** nodes to generate deep Week-over-Week (WoW) comparisons.<br>**⚠️ Required: Set your Property ID**<br>Open each of the 14 nodes → Property ID field → replace {YOUR_PROPERTY_ID} with your GA4 numeric property ID.<br>Found at: GA4 Admin → Property Settings → Property ID<br>**Node breakdown (Each has a Current & Previous week version):**<br>- **Overview** — totals for sessions, users, bounce rate, etc.<br>- **Top 5 Pages** — screens/pages by views<br>- **Top 5 Referrals** — traffic sources by sessions<br>- **Top 5 Events** — user interactions by count<br>- **Top 5 Countries** — geographic breakdown by sessions<br>- **Device Breakdown** — mobile / desktop / tablet split<br>- **New vs Returning** — audience loyalty breakdown<br>**All nodes have Execute Once = ON** to prevent duplicate rows. |
| Wait for All GA4 Data | n8n-nodes-base.merge | Synchronization barrier for all GA4 branches | GA4 - Overview Previous Week; GA4 - Top 5 Pages Previous Week; GA4 - Top 5 Referrals Previous Week; GA4 - Top 5 Events Previous Week; GA4 - Top 5 Countries Previous Week; GA4 - Device Breakdown Previous Week; GA4 - New vs Returning Previous Week | Generate AI Summary | ## 🔀 Merge<br>Waits for **all GA4 data chains (14 nodes total)** to complete before passing data forward.<br>Without this node, Gemini and Code nodes could trigger before all GA4 data is ready - causing empty or undefined values in the report.<br>Mode: Combine by Position |
| Generate AI Summary | @n8n/n8n-nodes-langchain.googleGemini | Generates AI executive summary from all WoW analytics data | Wait for All GA4 Data | Build Report & Email HTML | ## 🤖 Gemini — AI Executive Summary<br>Generates a **3-paragraph executive summary** covering overall performance, audience behaviour, and actionable recommendations. It analyzes the full Week-over-Week dataset from all 14 GA4 nodes to spot trends.<br>**Credential required:** Google Gemini API<br>Get your key at: [https://aistudio.google.com](https://aistudio.google.com)<br>**Model:** gemini-3.1-flash-lite-preview<br>If Gemini fails or returns empty, the report still sends normally — the AI section is hidden from the email automatically. |
| Build Report & Email HTML | n8n-nodes-base.code | Computes WoW changes and builds full HTML email output | Generate AI Summary | Send Weekly Report | ## 🧮 Code Node — Data Processing + Email Builder<br>Does 4 things:<br>1. Pulls data from all 14 GA4 nodes and Gemini by node name.<br>2. Calculates week-over-week % changes for **every single section** (Overview, Pages, Sources, Events, Countries, Devices, New vs Returning).<br>3. Builds the full HTML email with inline CSS and alignment fixes.<br>4. Outputs subject, html, and recipients for the Email node.<br>**To change recipients:**<br>Find `recipients:` near the bottom of the code → update the email address.<br>Multiple: `'email1@gmail.com,email2@gmail.com'`<br>**To change client name in footer:**<br>Find `AppStoneLab Technologies` in the HTML and replace with your client's brand name. |
| Send Weekly Report | n8n-nodes-base.emailSend | Sends the generated HTML report by email | Build Report & Email HTML |  | ## 📧 Send Email<br>Sends the HTML report to all recipients defined in the Code node.<br>**Fields mapped from Code node output:**<br>- To → {{ $json.recipients }}<br>- Subject → {{ $json.subject }}<br>- HTML Body → {{ $json.html }}<br>**Credential required:** Gmail OAuth2 or SMTP<br>For SMTP: update host, port, and login credentials in the credential settings. |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas documentation note |  |  |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Canvas documentation note for GA4 section |  |  |  |
| Sticky Note1 | n8n-nodes-base.stickyNote | Canvas documentation note for schedule section |  |  |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Canvas documentation note for merge section |  |  |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas documentation note for Gemini section |  |  |  |
| Sticky Note5 | n8n-nodes-base.stickyNote | Canvas documentation note for code section |  |  |  |
| Sticky Note6 | n8n-nodes-base.stickyNote | Canvas documentation note for email section |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it something like `Weekly GA4 Analytics Report For Website/App`.

2. **Set workflow timezone** in workflow settings before building date-sensitive logic.  
   This matters because all GA4 date expressions use `$now` and the schedule runs at a fixed weekday/hour.

3. **Add a Schedule Trigger node** named `Weekly Monday Trigger`.
   - Type: Schedule Trigger
   - Rule:
     - Interval field: weeks
     - Trigger day: Monday
     - Trigger hour: 8
   - This will be the only entry point.

4. **Create the first GA4 node** named `GA4 - Overview Current Week`.
   - Type: Google Analytics
   - Credential: Google Analytics OAuth2
   - Property ID: your GA4 numeric property ID
   - Date Range: Custom
   - Start Date: `{{$now.minus({days: 7}).startOf('day').toFormat('yyyy-MM-dd')}}`
   - End Date: `{{$now.minus({days: 1}).startOf('day').toFormat('yyyy-MM-dd')}}`
   - Metrics:
     - totalUsers
     - sessions
     - custom alias `bounce_rate` with expression `bounceRate`
     - custom alias `average_session_dur` with expression `averageSessionDuration`
     - custom alias `new_users` with expression `newUsers`
   - Execute Once: enabled

5. **Create the paired node** `GA4 - Overview Previous Week`.
   - Same settings as above except:
   - Start Date: `{{$now.minus({days: 14}).startOf('day').toFormat('yyyy-MM-dd')}}`
   - End Date: `{{$now.minus({days: 8}).startOf('day').toFormat('yyyy-MM-dd')}}`

6. **Connect** `Weekly Monday Trigger → GA4 - Overview Current Week → GA4 - Overview Previous Week`.

7. **Create `GA4 - Top 5 Pages`**.
   - Type: Google Analytics
   - Credential: same GA4 OAuth2
   - Property ID: same property
   - Date Range: Custom
   - Start: current week expression
   - End: current week expression
   - Limit: 5
   - Dimension: `unifiedScreenName`
   - Metrics:
     - custom alias `screen_page_view` with expression `screenPageViews`
     - custom alias `average_session_duration` with expression `averageSessionDuration`
   - Execute Once: enabled

8. **Create `GA4 - Top 5 Pages Previous Week`** with the same metrics/dimension and previous-week dates.

9. **Connect** `Weekly Monday Trigger → GA4 - Top 5 Pages → GA4 - Top 5 Pages Previous Week`.

10. **Create `GA4 - Top 5 Referrals`**.
    - Type: Google Analytics
    - Limit: 5
    - Metric: sessions
    - Dimension: `sessionSourceMedium`
    - Order by metric sessions descending
    - Current-week dates
    - Execute Once: enabled

11. **Create `GA4 - Top 5 Referrals Previous Week`** with identical structure and previous-week dates.

12. **Connect** `Weekly Monday Trigger → GA4 - Top 5 Referrals → GA4 - Top 5 Referrals Previous Week`.

13. **Create `GA4 - Top 5 Events`**.
    - Type: Google Analytics
    - Limit: 5
    - Dimension: `eventName`
    - Metric: custom alias `event_count` with expression `eventCount`
    - Order descending by event count
    - Current-week dates
    - Execute Once: enabled

14. **Create `GA4 - Top 5 Events Previous Week`** using the previous-week dates.

15. **Connect** `Weekly Monday Trigger → GA4 - Top 5 Events → GA4 - Top 5 Events Previous Week`.

16. **Create `GA4 - Top 5 Countries`**.
    - Type: Google Analytics
    - Limit: 5
    - Dimension: `country`
    - Metric: sessions
    - Order descending by sessions
    - Current-week dates
    - Execute Once: enabled

17. **Create `GA4 - Top 5 Countries Previous Week`** with previous-week dates.

18. **Connect** `Weekly Monday Trigger → GA4 - Top 5 Countries → GA4 - Top 5 Countries Previous Week`.

19. **Create `GA4 - Device Breakdown`**.
    - Type: Google Analytics
    - Dimension: `deviceCategory`
    - Metric: sessions
    - Order descending by sessions
    - Current-week dates
    - Execute Once: enabled

20. **Create `GA4 - Device Breakdown Previous Week`** with previous-week dates.

21. **Connect** `Weekly Monday Trigger → GA4 - Device Breakdown → GA4 - Device Breakdown Previous Week`.

22. **Create `New vs Returning Users Breakdown`**.
    - Type: Google Analytics
    - Dimension: `newVsReturning`
    - Metrics:
      - sessions
      - totalUsers
    - Current-week dates
    - Execute Once: enabled

23. **Create `GA4 - New vs Returning Previous Week`** with previous-week dates.

24. **Connect** `Weekly Monday Trigger → New vs Returning Users Breakdown → GA4 - New vs Returning Previous Week`.

25. **Add a Merge node** named `Wait for All GA4 Data`.
    - Type: Merge
    - Mode: Combine
    - Combine By: Position
    - Number of Inputs: 7
    - Include Unpaired: true

26. **Connect each previous-week node into the Merge node**:
    - `GA4 - Overview Previous Week` → input 0
    - `GA4 - Top 5 Pages Previous Week` → input 1
    - `GA4 - Top 5 Referrals Previous Week` → input 2
    - `GA4 - Top 5 Events Previous Week` → input 3
    - `GA4 - Top 5 Countries Previous Week` → input 4
    - `GA4 - Device Breakdown Previous Week` → input 5
    - `GA4 - New vs Returning Previous Week` → input 6

27. **Add a Google Gemini node** named `Generate AI Summary`.
    - Type: Google Gemini
    - Credential: Google Gemini / PaLM API credential
    - Model: `models/gemini-3.1-flash-lite-preview`
    - Add one message containing a prompt that:
      - identifies the model as a professional digital analytics consultant
      - injects all current-week and previous-week values using node references
      - asks for exactly 3 to 5 bullets
      - requires dash-prefixed bullets
      - requires one sentence per bullet, max 25 words
      - forbids headers and bold formatting
    - Use node references exactly if you keep the same node names, because the Code node also depends on those names.

28. **Connect** `Wait for All GA4 Data → Generate AI Summary`.

29. **Add a Code node** named `Build Report & Email HTML`.

30. **Paste JavaScript into the Code node** that performs all of the following:
    - Reads all GA4 node results with `$('<node name>').first()` and `.all()`
    - Reads Gemini output defensively from multiple possible response fields
    - Normalizes empty screen names to `(empty)`
    - Renames `(direct) / (none)` to `Direct / App Open`
    - Filters out technical events like `first_visit`, `first_open`, and `page_view`
    - Computes WoW deltas
    - Applies inverse-good logic for bounce rate
    - Builds a full HTML email string with inline CSS
    - Returns JSON with:
      - `subject`
      - `html`
      - `recipients`
      - `_debug`

31. **Inside the Code node, set recipients** in the returned JSON, for example:
    - `recipients: 'user@example.com'`
    - For multiple recipients, use comma-separated emails in a single string.

32. **Inside the Code node, optionally customize branding**.
    - Replace `AppStoneLab Technologies`
    - Replace `https://appstonelab.com`
    - Adjust footer text and unsubscribe link placeholder

33. **Connect** `Generate AI Summary → Build Report & Email HTML`.

34. **Add an Email Send node** named `Send Weekly Report`.
    - Type: Email Send
    - Credential: SMTP
    - Subject: `{{$json.subject}}`
    - HTML: `{{$json.html}}`
    - To: `{{$json.recipients}}, jignesh.sanghani@dev.appstonelab.com`
      - Change this if you do not want the hard-coded extra recipient.
    - From: `{{$json.recipients}}`
      - Recommended improvement: replace this with a fixed sender allowed by your SMTP server, such as `reports@yourdomain.com`.

35. **Connect** `Build Report & Email HTML → Send Weekly Report`.

36. **Create credentials**:
    - **Google Analytics OAuth2**
      - Must have access to the selected GA4 property
    - **Google Gemini API**
      - API key from [https://aistudio.google.com](https://aistudio.google.com)
    - **SMTP**
      - Host, port, username, password, secure/TLS settings according to your mail provider

37. **Test each GA4 node individually** before running the whole workflow.
    - Confirm field names are returned as expected:
      - `totalUsers`
      - `sessions`
      - `new_users`
      - `bounce_rate`
      - `average_session_dur`
      - `screen_page_view`
      - `event_count`
      - etc.

38. **Test the Gemini node**.
    - Confirm it returns text.
    - If it fails, the workflow can still send email, but the AI summary section will be omitted.

39. **Run the full workflow manually**.
    - Check the Code node output for:
      - valid subject
      - populated html
      - correct recipients
      - `_debug` indicators

40. **Validate email rendering** in your email client.
    - Pay attention to:
      - table alignment
      - mobile display
      - font fallbacks
      - spam filtering
      - sender authorization

41. **Activate the workflow** once verified.

### Rebuild Constraints and Important Implementation Notes
- The node names matter because the Gemini prompt and Code node reference them directly.
- If you rename nodes, you must update every expression that uses `$('<node name>')`.
- The workflow has only one entry point and no sub-workflow nodes.
- There are no Execute Workflow / sub-workflow dependencies to configure.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Automatically runs every Monday at 8:00 AM and delivers a formatted HTML performance report to stakeholders via email. | Workflow purpose |
| Before using the workflow, set the GA4 Property ID in all 14 GA4 nodes. | GA4 configuration |
| Connect Google Analytics OAuth2, Gemini API, and Gmail/SMTP credentials before activation. | Credential setup |
| Set recipient email addresses either in the Code node output or the Email node. | Delivery setup |
| Set the workflow timezone in workflow settings to avoid date-range drift. | Scheduling/date accuracy |
| Gemini API key source | [https://aistudio.google.com](https://aistudio.google.com) |
| Client branding shown in the generated email footer | [https://appstonelab.com](https://appstonelab.com) |
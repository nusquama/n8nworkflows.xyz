Analyze WooCommerce category sales over time with Airtable and Slack

https://n8nworkflows.xyz/workflows/analyze-woocommerce-category-sales-over-time-with-airtable-and-slack-13840


# Analyze WooCommerce category sales over time with Airtable and Slack

This document provides a comprehensive technical analysis and reconstruction guide for the **Automated Category Sales Performance Report** workflow in n8n.

---

### 1. Workflow Overview
The primary purpose of this workflow is to automate the analysis of WooCommerce sales data by product category. It compares current performance against a previous period (daily, weekly, or monthly) and classifies categories based on their revenue contribution and growth. 

The workflow is organized into four logical phases:
1.  **Configuration & Timing:** Defines reporting rules and calculates UTC date ranges for comparison.
2.  **Data Extraction & Normalization:** Retrieves raw orders and product data from WooCommerce and flattens them into granular line-item records.
3.  **Analytics Engine:** Aggregates data by category, calculates market share and growth percentage, and applies business logic to rank performance.
4.  **Distribution:** Logs detailed metrics to Airtable and pushes a high-level executive summary to Slack.

---

### 2. Block-by-Block Analysis

#### 2.1 Configuration & Timing
This block establishes the parameters for the report, such as the look-back window and performance thresholds.
*   **Nodes Involved:** `Run On Schedule`, `Report Configuration`, `Calculate Reporting Periods`.
*   **Logic:** 
    *   The `Report Configuration` node sets variables like `granularity` (days, weekly, monthly) and growth thresholds (e.g., `momentum_pct`).
    *   `Calculate Reporting Periods` uses JavaScript to generate ISO strings for the start and end of both the current and previous periods to ensure consistent filtering in the API calls.
*   **Edge Cases:** Errors in UTC date math; unsupported granularity strings.

#### 2.2 Data Extraction & Normalization
Fetches raw data and transforms it into a unified format for analysis.
*   **Nodes Involved:** `Fetch Orders — Current Period`, `Fetch Orders — Previous Period`, `Fetch Products`, `Normalize Order Items — Current`, `Normalize Order Items — Previous`, `Category Map`.
*   **Logic:**
    *   Orders are filtered by status (`completed`) and date.
    *   `Normalize` nodes iterate through `line_items` or `fee_lines`, ensuring that even orders with multiple products are counted correctly at the item level.
    *   `Category Map` creates a lookup object from the `Fetch Products` node to link specific product IDs to their primary category names.
*   **Edge Cases:** WooCommerce API timeouts; products without categories; orders containing only fees.

#### 2.3 Analytics Engine
Performs the mathematical heavy lifting.
*   **Nodes Involved:** `Combine Orders & Products`, `Aggregate Sales by Category`, `Classify Category Performance`.
*   **Logic:**
    *   `Aggregate Sales by Category` calculates `revenue_current`, `revenue_prev`, `abs_change`, and `share_of_total` (category revenue / total revenue).
    *   `Classify Category Performance` ranks categories by revenue. It uses percentile thresholds (Top 20% = Top Performer, Top 50% = Momentum, etc.) and assigns recommended actions (e.g., "Feature on homepage").
*   **Edge Cases:** Division by zero if the previous period had no sales; "n/a" handling for new categories.

#### 2.4 Distribution
Handles data persistence and stakeholder notification.
*   **Nodes Involved:** `Prepare Fields for Reporting`, `Save Results to Airtable`, `Build Slack Summary`, `Send Report to Slack`.
*   **Logic:**
    *   Formats the calculated data for Airtable's schema.
    *   `Build Slack Summary` generates a Markdown-formatted message highlighting top categories and those needing attention (the "At Risk" segment).
*   **Edge Cases:** Airtable rate limits; Slack channel ID mismatches.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Run On Schedule | Schedule Trigger | Trigger | None | Report Configuration | Starts the workflow automatically on a fixed schedule (daily / weekly). |
| Report Configuration | Set | Config | Run On Schedule | Calculate Reporting Periods | Sets the report rules and automatically calculates current and previous date ranges for sales comparison. |
| Calculate Reporting Periods | Code | Logic | Report Configuration | Fetch Orders (Curr/Prev), Fetch Products | Sets the report rules and automatically calculates current and previous date ranges for sales comparison. |
| Fetch Orders — Current Period | WooCommerce | Data In | Calculate Reporting Periods | Normalize Order Items — Current | Fetch Orders — Current Period: Fetches completed WooCommerce orders from the current time period. |
| Fetch Orders — Previous Period | WooCommerce | Data In | Calculate Reporting Periods | Normalize Order Items — Previous | Fetch Orders — Previous Period: Fetches completed WooCommerce orders from the previous comparison period. |
| Fetch Products | WooCommerce | Data In | Calculate Reporting Periods | Category Map | Fetch Products: Loads product and category details from WooCommerce. |
| Category Map | Code | Transform | Fetch Products | Combine Orders & Products | Category Map: Maps each product to its category for accurate category reporting. |
| Normalize Order Items — Current | Code | Transform | Fetch Orders — Current Period | Combine Orders & Products | Normalize Order Items — Current: Breaks current orders into individual product-level sales rows. |
| Normalize Order Items — Previous | Code | Transform | Fetch Orders — Previous Period | Combine Orders & Products | Normalize Order Items — Previous: Breaks previous orders into individual product-level sales rows. |
| Combine Orders & Products | Merge | Utility | Normalize (Curr/Prev), Category Map | Aggregate Sales by Category | Combine Orders & Products: Combines current and previous sales data into a single stream. |
| Aggregate Sales by Category | Code | Analytics | Combine Orders & Products | Classify Category Performance | Aggregate Sales by Category: Calculates total revenue, units sold and orders per category. |
| Classify Category Performance | Code | Analytics | Aggregate Sales by Category | Prepare Fields for Reporting | Classify Category Performance: Labels categories as Top Performer, Momentum, Steady or At Risk. |
| Prepare Fields for Reporting | Set | Transform | Classify Category Performance | Save Results to Airtable | Prepare Fields for Reporting: Formats and selects final fields needed for reports and storage. |
| Save Results to Airtable | Airtable | Data Out | Prepare Fields for Reporting | Build Slack Summary | Save Results to Airtable: Saves category performance results to Airtable for tracking and history. |
| Build Slack Summary | Code | Transform | Save Results to Airtable | Send Report to Slack | Build Slack Summary: Creates a short, readable summary of category performance. |
| Send Report to Slack | Slack | Data Out | Build Slack Summary | None | Send Report to Slack: Sends the final performance summary to the Slack channel. |

---

### 4. Reproducing the Workflow from Scratch

1.  **Trigger:** Add a **Schedule Trigger**. Set the interval to "Daily" or "Weekly" depending on your reporting needs.
2.  **Configuration (Set Node):** Create a node named `Report Configuration`. Add fields: `granularity` (string: "days"), `top_categories_limit` (number: 8), and various threshold percentages (e.g., `steady_band`: 10).
3.  **Date Calculation (Code Node):** Add a Code node. Use JavaScript to calculate `period_start`, `period_end`, `prev_period_start`, and `prev_period_end`.
    *   *Tip:* Use `new Date(Date.UTC(...))` to avoid timezone shifts during calculation.
4.  **WooCommerce Integration:**
    *   Create two WooCommerce nodes for **Orders** (Operation: Get All). Use expressions to map the "After" and "Before" filters to the dates calculated in step 3. Set status to `completed`.
    *   Create one WooCommerce node for **Products** (Operation: Get All) to fetch metadata.
5.  **Normalization (Code Nodes):** 
    *   Write a script for `Normalize Order Items` that loops through the `line_items` array of every order and returns a new array of objects containing `item_id`, `item_total`, and `order_date`.
    *   Create the `Category Map` node to transform the product list into an object keyed by `product_id`.
6.  **Merging:** Use a **Merge** node with 3 inputs (Wait: All inputs to arrive) to synchronize the data streams.
7.  **Analytics Logic (Code Nodes):**
    *   **Aggregate:** Sum `item_total` by category ID. Calculate the difference between current and previous periods.
    *   **Classify:** Implement sorting logic. Use `array.sort` on revenue and assign tags based on the index position relative to the total number of categories.
8.  **Output Setup:**
    *   **Airtable:** Configure a "Create" operation. Map your calculated variables to the corresponding column names in your Airtable base.
    *   **Slack:** Add a Code node to build a message string using template literals (e.g., ``*Top Categories* \n • ${cat_name}: ${revenue}``). Use a Slack node to post this string to a specific channel ID.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **How it Works** | Automatically analyzes sales data by product category and compares current performance with the previous period. |
| **Setup Step: WooCommerce** | Connect your order source for both current and previous periods using OAuth2 or API keys. |
| **Setup Step: Airtable** | Connect Airtable and map fields for category metrics and recommended actions. |
| **Performance Logic** | Classification is dynamic; it adapts based on the total number of categories with positive revenue. |
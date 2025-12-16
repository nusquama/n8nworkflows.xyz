Automate Shopify Inventory Reordering with Predictive Analytics and Google Sheets

https://n8nworkflows.xyz/workflows/automate-shopify-inventory-reordering-with-predictive-analytics-and-google-sheets-11799


# Automate Shopify Inventory Reordering with Predictive Analytics and Google Sheets

### 1. Workflow Overview

This workflow automates the inventory management and reordering process for a Shopify store by integrating Shopify data with Google Sheets and performing predictive analytics. It is designed to optimize inventory levels, reduce stockouts, and streamline supplier communications through automated Purchase Orders (POs).

**Target Use Cases:**
- Retailers or e-commerce operators using Shopify who want to automate inventory reordering.
- Businesses seeking to integrate multiple data sources (Shopify, Google Sheets) for inventory insights.
- Teams requiring notifications and operational controls around stock risks, supplier availability, and order approvals.

**Logical Blocks:**

- **1.1 Trigger & Configuration**  
  Hourly scheduled trigger to run the workflow and load configurable business parameters like store URL, reorder multipliers, safety stock days, budget limits, and thresholds.

- **1.2 Data Acquisition from Shopify & Google Sheets**  
  Parallel extraction of live inventory levels, product details, and recent orders from Shopify; and fetching Inventory Master, Suppliers list, and Purchase Order Log from Google Sheets.

- **1.3 Data Merging and Sales Velocity Calculation**  
  Combining all data sources into a unified SKU-level dataset and calculating sales velocity metrics (7-day and 30-day averages).

- **1.4 Reorder Point Calculation and Risk Assessment**  
  Computing dynamic reorder points per SKU based on demand and lead time, determining stockout risk, and triggering alerts for critical stock levels.

- **1.5 Multi-Warehouse Inventory Redistribution Logic**  
  Evaluating inventory across warehouses to identify transfer opportunities to prevent stockouts before placing new orders.

- **1.6 Supplier Data Enrichment and Availability Checks**  
  Matching SKUs with supplier records and verifying supplier availability to ensure only feasible items proceed.

- **1.7 Business Rules and Order Quantity Calculation**  
  Applying operational rules including business-day checks, Minimum Order Quantity (MOQ) compliance, promotional period handling, budget limits, and large order approval thresholds.

- **1.8 Profit Optimization and Prioritization**  
  Ranking SKUs by profit priority using margin, sales velocity, carrying costs, and stockout risk to optimize reorder decisions.

- **1.9 Purchase Order Structuring and Communication**  
  Structuring PO line items, grouping by supplier, preparing email contexts, sending POs via email and supplier API, then logging confirmations.

- **1.10 Analytics Sync and Notifications**  
  Outputting KPIs to dashboards, syncing with accounting systems, detecting slow-moving items with actionable recommendations, and sending Slack alerts and daily summaries.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Configuration

**Overview:**  
Starts the workflow every hour and sets all key configuration parameters allowing adjustments without modifying node logic.

**Nodes Involved:**  
- Hourly Trigger  
- Workflow Configuration  
- Sticky Note1 (commentary)

**Node Details:**  
- **Hourly Trigger**  
  - Type: Schedule Trigger  
  - Configured to run hourly (every 1 hour).  
  - No inputs; outputs trigger signal to Workflow Configuration.  
  - Potential issues: workflow execution delays if previous run is still active.

- **Workflow Configuration**  
  - Type: Set node  
  - Assigns static business parameters: Shopify store URL, reorder point multiplier (1.5), safety stock days (7), budget limit ($50,000), large order threshold ($10,000), slow mover threshold (90 days).  
  - Outputs these as JSON to downstream nodes for dynamic use.  
  - Edge case: missing or incorrect config values may cause calculation errors downstream.

---

#### 1.2 Data Acquisition from Shopify & Google Sheets

**Overview:**  
Retrieves inventory levels, product details, and recent orders from Shopify; loads inventory master data, suppliers, and purchase order logs from Google Sheets.

**Nodes Involved:**  
- Get Inventory Levels  
- Get Product Details  
- Get Last 30 Days Orders  
- Read Inventory Master  
- Read Suppliers  
- Read Purchase Order Log  
- Sticky Note2 (Shopify data)  
- Sticky Note3 (Google Sheets data)

**Node Details:**  
- **Get Inventory Levels**  
  - Shopify node fetching current inventory levels authenticated via access token.  
  - Outputs inventory items with stock quantities and locations.  
  - Failure modes: auth errors, Shopify API rate limits or downtime.

- **Get Product Details**  
  - Shopify node fetching all product variants and metadata with SKU and inventory_item_id mappings.  
  - Used to cross-reference SKU and inventory item info.

- **Get Last 30 Days Orders**  
  - Shopify node fetching all orders created within last 30 days.  
  - Provides order line items with SKUs and quantities for sales velocity calculations.

- **Read Inventory Master, Read Suppliers, Read Purchase Order Log**  
  - Google Sheets nodes reading configured sheets by name and document ID (from environment variables or placeholders).  
  - Service account authentication is used.  
  - Provide business context (reorder points, suppliers, past orders) enriching Shopify data.

- Edge cases: missing or invalid Google Sheet IDs, access permission issues, empty sheets.

---

#### 1.3 Data Merging and Sales Velocity Calculation

**Overview:**  
Combines all Shopify and Google Sheets data sources into a single dataset and calculates sales velocity metrics per SKU.

**Nodes Involved:**  
- Merge All Data Sources  
- Calculate Sales Velocity  
- Sticky Note4 (merge explanation)  
- Sticky Note5 (sales velocity explanation)

**Node Details:**  
- **Merge All Data Sources**  
  - Merge node combining inputs from all six data sources by position to create unified records per SKU.

- **Calculate Sales Velocity**  
  - Code node parsing merged data arrays.  
  - Extracts orders, inventory, products, inventory master, and suppliers into maps keyed by SKU or IDs.  
  - Calculates total sales over 7 and 30 days and averages daily sales.  
  - Outputs enriched SKU objects including sales velocity and last sale date.  
  - Edge cases: SKUs missing from some sources, null or malformed sales data.

---

#### 1.4 Reorder Point Calculation and Risk Assessment

**Overview:**  
Calculates SKU-specific dynamic reorder points based on sales velocity, lead time, and safety stock; evaluates if reorder is needed and assesses stockout risk.

**Nodes Involved:**  
- Calculate Dynamic Reorder Point  
- Check Reorder Point Reached  
- Calculate Stockout Risk  
- Check High Stockout Risk  
- Alert Critical Stock Risk  
- Sticky Note6 (dynamic reorder point)  
- Sticky Note7 (stockout risk & alerts)

**Node Details:**  
- **Calculate Dynamic Reorder Point**  
  - Calculates reorder point = avg_daily_sales * lead_time + safety_stock (where safety_stock = avg_daily_sales * safetyStockDays from config).  
  - Determines if current stock <= reorder point to flag reorder need.  
  - Calculates days until stockout and recommended order quantity.

- **Check Reorder Point Reached**  
  - If current stock <= reorder point, passes SKU for stockout risk calculation.

- **Calculate Stockout Risk**  
  - Calculates risk level (0.1 to 0.9) based on days until stockout relative to lead time.

- **Check High Stockout Risk**  
  - If stockout risk > 0.7, triggers critical alert path.

- **Alert Critical Stock Risk**  
  - Sends Slack alert to configured channel with SKU and stock details.

- Edge cases: zero or missing sales velocity causing division by zero, lead time missing defaults to 7 days, Slack webhook failures.

---

#### 1.5 Multi-Warehouse Inventory Redistribution Logic

**Overview:**  
Analyzes stock levels across warehouses to identify overstock and understock situations, recommending inter-warehouse transfers to avoid new orders.

**Nodes Involved:**  
- Check Warehouse Redistribution Possible  
- Multi-Warehouse Distribution Logic  
- Structure Transfer Recommendations  
- Alert Critical Stock Risk (for transfer alerts)  
- Sticky Note8 (warehouse redistribution)

**Node Details:**  
- **Check Warehouse Redistribution Possible**  
  - Conditional node verifying presence of warehouse overstock and understock data.

- **Multi-Warehouse Distribution Logic**  
  - Code node grouping inventory by SKU and warehouse.  
  - Identifies transfer opportunities by comparing overstock and understock quantities and calculates safe transfer amounts.  
  - Outputs transfer recommendations prioritized by urgency.

- **Structure Transfer Recommendations**  
  - Formats transfer data with transfer ID, warehouses, quantity, and timestamps.

- Edge cases: warehouses missing data, transfer quantity zero, circular transfers.

---

#### 1.6 Supplier Data Enrichment and Availability Checks

**Overview:**  
Enhances SKU data with supplier information, verifies supplier availability, lead times, MOQs, and unit costs for downstream ordering logic.

**Nodes Involved:**  
- Check Supplier Availability  
- Enrich with Supplier Data  
- Sticky Note9 (supplier checks)

**Node Details:**  
- **Check Supplier Availability**  
  - Conditional node passing only SKUs with supplier availability flag true.

- **Enrich with Supplier Data**  
  - Code node builds a supplier lookup map with multiple ID variations.  
  - Merges supplier fields (name, email, API endpoint, lead time, MOQ, unit cost) into SKU data.  
  - Defaults lead time to 7 days, MOQ to 1 if missing.

- Edge cases: supplier data missing, inconsistent supplier IDs, false availability flags.

---

#### 1.7 Business Rules and Order Quantity Calculation

**Overview:**  
Applies operational constraints such as business day restrictions, MOQ enforcement, promotional period checking, budget limits, and large order approval gating.

**Nodes Involved:**  
- Check Business Day  
- Check MOQ Met  
- Check Promotional Period  
- Check Budget Limit  
- Check Large Order Approval Needed  
- Calculate Order Quantity and Value  
- Sticky Note13 (business rules)  
- Sticky Note14 (order calculation & prioritization)

**Node Details:**  
- **Check Business Day**  
  - Runs only on weekdays (Monday-Friday).

- **Check MOQ Met**  
  - Ensures order quantity >= supplier MOQ.

- **Check Promotional Period**  
  - Conditional node to detect if SKU is in promotional period (flag in data).

- **Check Budget Limit**  
  - Verifies total purchase order value remains under configured budget.

- **Check Large Order Approval Needed**  
  - Flags orders exceeding large order threshold for additional review.

- **Calculate Order Quantity and Value**  
  - Adjusts order quantity to meet MOQ; calculates total PO value and timestamps.

- Edge cases: incorrect date/time causing business day misfires, missing MOQ or promotional flags.

---

#### 1.8 Profit Optimization and Prioritization

**Overview:**  
Scores SKUs by profitability using margin, velocity, carrying cost, and stockout risk; ranks and prioritizes SKUs for ordering under budget constraints.

**Nodes Involved:**  
- Optimize Profit Priority  
- Sticky Note14 (profit optimization details)

**Node Details:**  
- **Optimize Profit Priority**  
  - Code node calculating margin = (unit price - cost)/price, profit score = (margin * velocity) / (carrying cost + stockout risk).  
  - Sorts SKUs descending by profit score.  
  - Assigns priority tiers (High, Medium, Low).

- Edge cases: missing cost or price data leading to zero margin, zero velocity SKUs scored zero profit priority.

---

#### 1.9 Purchase Order Structuring and Communication

**Overview:**  
Structures PO line items, groups by supplier, prepares email contexts, sends POs via Gmail and supplier API, waits for confirmation, and logs PO details.

**Nodes Involved:**  
- Structure PO Line Items  
- Prepare PO Email Context  
- Send PO Email  
- Send PO to Supplier API  
- Wait for PO Confirmation  
- Update Purchase Order Log  
- Alert PO Sent (Slack notification)  
- Sticky Note15 (PO line items & grouping)  
- Sticky Note16 (send PO & notify)  
- Sticky Note17 (PO confirmation & logging)

**Node Details:**  
- **Structure PO Line Items**  
  - Sets PO line item fields including IDs, supplier IDs, SKUs, quantities, unit and total prices.

- **Prepare PO Email Context**  
  - Groups line items by supplier, calculates totals, generates PO numbers, formats line items as HTML tables, adds delivery instructions.

- **Send PO Email**  
  - Gmail node sending formatted PO email to supplier address.  
  - Uses service account authentication.  
  - Failure modes: invalid email, authentication issues.

- **Send PO to Supplier API**  
  - HTTP POST to supplier API endpoint with PO data and authorization header.  
  - Safe error handling enabled.

- **Wait for PO Confirmation**  
  - Waits 1 hour for confirmation webhook (manual or API).  

- **Update Purchase Order Log**  
  - Appends or updates PO log in Google Sheets with PO status, timestamps, and supplier info.

- **Alert PO Sent**  
  - Sends Slack notification of sent PO.

- Edge cases: missing supplier emails or API endpoints, delayed confirmations, Google Sheets update conflicts.

---

#### 1.10 Analytics Sync and Notifications

**Overview:**  
Structures analytics KPIs, writes metrics to Google Sheets dashboards, syncs financial data to accounting system, detects slow-moving products with recommendations, and sends Slack alerts and daily summaries.

**Nodes Involved:**  
- Structure Analytics Data  
- Write Dashboard Metrics  
- Write Scenario Planning  
- Sync to Accounting System  
- Detect Slow-Movers  
- Structure Slow-Mover Data  
- Alert Slow-Mover Suggestions  
- Aggregate Daily Summary  
- Send Daily Summary (Slack)  
- Sticky Note11 (slow-movers detection)  
- Sticky Note12 (analytics & system sync)

**Node Details:**  
- **Structure Analytics Data**  
  - Calculates inventory turnover, carrying cost, stockout saves, timestamp for each SKU.

- **Write Dashboard Metrics & Write Scenario Planning**  
  - Append or update Google Sheets with KPI data for dashboard visualization and scenario planning.

- **Sync to Accounting System**  
  - HTTP POST with financial KPIs to external accounting API using generic HTTP header authentication.

- **Detect Slow-Movers**  
  - Identifies SKUs with velocity below threshold, calculates days since last sale, suggests discounts or bundle partners.

- **Structure Slow-Mover Data**  
  - Formats slow-mover info for alerts.

- **Alert Slow-Mover Suggestions**  
  - Sends Slack notification for slow-moving SKUs with suggested actions.

- **Aggregate Daily Summary**  
  - Safely aggregates counts and totals of POs, alerts, and slow-movers.

- **Send Daily Summary**  
  - Posts summary message to Slack channel.

- Edge cases: missing or invalid KPI data, Slack or API failures, empty slow-mover list.

---

### 3. Summary Table

| Node Name                      | Node Type                 | Functional Role                                | Input Node(s)                               | Output Node(s)                                  | Sticky Note                                                                                          |
|--------------------------------|---------------------------|------------------------------------------------|---------------------------------------------|------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Hourly Trigger                 | Schedule Trigger          | Triggers workflow hourly                       |                                             | Workflow Configuration                         | ## Trigger & Configuration<br>Runs hourly and loads all business parameters...                      |
| Workflow Configuration         | Set                       | Loads configurable parameters                  | Hourly Trigger                              | Get Inventory Levels, Get Product Details, Get Last 30 Days Orders, Read Suppliers, Read Purchase Order Log, Read Inventory Master | ## Trigger & Configuration                                                                        |
| Get Inventory Levels           | Shopify                   | Fetch current inventory levels                 | Workflow Configuration                      | Merge All Data Sources                         | ## Shopify Data Sources                                                                           |
| Get Product Details            | Shopify                   | Fetch all product and variant details          | Workflow Configuration                      | Merge All Data Sources                         | ## Shopify Data Sources                                                                           |
| Get Last 30 Days Orders        | Shopify                   | Fetch recent order history                      | Workflow Configuration                      | Merge All Data Sources                         | ## Shopify Data Sources                                                                           |
| Read Inventory Master          | Google Sheets             | Read inventory master sheet                     | Workflow Configuration                      | Merge All Data Sources                         | ## Google Sheets Data Sources                                                                     |
| Read Suppliers                | Google Sheets             | Read supplier directory                         | Workflow Configuration                      | Merge All Data Sources                         | ## Google Sheets Data Sources                                                                     |
| Read Purchase Order Log        | Google Sheets             | Read past purchase order records                | Workflow Configuration                      | Merge All Data Sources                         | ## Google Sheets Data Sources                                                                     |
| Merge All Data Sources         | Merge                     | Combine all data sources by position            | Get Inventory Levels, Get Product Details, Get Last 30 Days Orders, Read Inventory Master, Read Suppliers, Read Purchase Order Log | Calculate Sales Velocity                      | ## Merge All Data                                                                                |
| Calculate Sales Velocity       | Code                      | Calculate sales velocity & averages             | Merge All Data Sources                      | Calculate Dynamic Reorder Point, Detect Slow-Movers | ## Sales Velocity                                                                              |
| Calculate Dynamic Reorder Point| Code                      | Calculate reorder points & reorder flags        | Calculate Sales Velocity                    | Check Reorder Point Reached, Structure Inventory Updates, Structure Analytics Data | ## Dynamic Reorder Point                                                                        |
| Check Reorder Point Reached    | If                        | Check if stock <= reorder point                  | Calculate Dynamic Reorder Point             | Calculate Stockout Risk, Structure Analytics Data | ## Dynamic Reorder Point                                                                        |
| Calculate Stockout Risk        | Code                      | Compute stockout risk score                       | Check Reorder Point Reached                 | Check High Stockout Risk                       | ## Stockout Risk & Critical Alerts                                                              |
| Check High Stockout Risk       | If                        | Identify critical stockout risk                   | Calculate Stockout Risk                      | Check Warehouse Redistribution Possible, Alert Critical Stock Risk | ## Stockout Risk & Critical Alerts                                                              |
| Alert Critical Stock Risk      | Slack                     | Send Slack alert for critical stock risks       | Check High Stockout Risk, Structure Transfer Recommendations |                                              | ## Stockout Risk & Critical Alerts                                                              |
| Check Warehouse Redistribution Possible | If               | Detect warehouse overstock and understock       | Check High Stockout Risk                     | Multi-Warehouse Distribution Logic, Check Supplier Availability | ## Warehouse Redistribution                                                                    |
| Multi-Warehouse Distribution Logic | Code                  | Analyze warehouses for transfer recommendations | Check Warehouse Redistribution Possible     | Structure Transfer Recommendations            | ## Warehouse Redistribution                                                                    |
| Structure Transfer Recommendations | Set                    | Format warehouse transfer recommendations        | Multi-Warehouse Distribution Logic          | Alert Critical Stock Risk                      | ## Warehouse Redistribution                                                                    |
| Check Supplier Availability   | If                        | Filter SKUs with available suppliers             | Check Warehouse Redistribution Possible     | Enrich with Supplier Data                      | ## Supplier Checks                                                                             |
| Enrich with Supplier Data     | Code                      | Add supplier info and flags to SKUs              | Check Supplier Availability                  | Check Business Day                             | ## Supplier Checks                                                                             |
| Check Business Day            | If                        | Ensure orders only on weekdays                    | Enrich with Supplier Data                    | Check MOQ Met                                  | ## Business Rules & Approval Gates                                                            |
| Check MOQ Met                | If                        | Verify Minimum Order Quantity compliance          | Check Business Day                           | Check Promotional Period                        | ## Business Rules & Approval Gates                                                            |
| Check Promotional Period      | If                        | Detect promotional period status                   | Check MOQ Met                               | Check Budget Limit                             | ## Business Rules & Approval Gates                                                            |
| Check Budget Limit            | If                        | Enforce budget constraints                         | Check Promotional Period                     | Check Large Order Approval Needed               | ## Business Rules & Approval Gates                                                            |
| Check Large Order Approval Needed | If                    | Flag orders exceeding large order threshold       | Check Budget Limit                           | Calculate Order Quantity and Value              | ## Business Rules & Approval Gates                                                            |
| Calculate Order Quantity and Value | Code                  | Adjust order quantity & calculate PO value        | Check Large Order Approval Needed            | Optimize Profit Priority                        | ## Order Calculation & Profit Prioritization                                                  |
| Optimize Profit Priority      | Code                      | Rank SKUs by profit priority                        | Calculate Order Quantity and Value           | Structure PO Line Items                         | ## Order Calculation & Profit Prioritization                                                  |
| Structure PO Line Items      | Set                       | Format PO line items                                | Optimize Profit Priority                      | Prepare PO Email Context                        | ## PO Line Items & Supplier Grouping                                                        |
| Prepare PO Email Context      | Code                      | Group by supplier and prepare email content         | Structure PO Line Items                       | Send PO Email, Send PO to Supplier API          | ## PO Line Items & Supplier Grouping                                                        |
| Send PO Email                | Gmail                     | Send purchase order email                            | Prepare PO Email Context                      | Wait for PO Confirmation, Alert PO Sent         | ## Send PO & Notify Team                                                                    |
| Send PO to Supplier API       | HTTP Request              | Send PO to supplier API endpoint                     | Prepare PO Email Context                      |                                                | ## Send PO & Notify Team                                                                    |
| Wait for PO Confirmation      | Wait                      | Wait for PO confirmation                             | Send PO Email                                | Update Purchase Order Log                       | ## PO Confirmation, Logging & Daily Summary                                               |
| Update Purchase Order Log     | Google Sheets             | Append or update PO log                              | Wait for PO Confirmation                      | Aggregate Daily Summary                         | ## PO Confirmation, Logging & Daily Summary                                               |
| Alert PO Sent                | Slack                     | Notify team of sent PO                                | Send PO Email                                |                                                | ## Send PO & Notify Team                                                                    |
| Aggregate Daily Summary       | Code                      | Aggregate daily metrics with error handling          | Update Purchase Order Log                     | Send Daily Summary                              | ## PO Confirmation, Logging & Daily Summary                                               |
| Send Daily Summary            | Slack                     | Post daily summary of POs and alerts                 | Aggregate Daily Summary                       |                                                | ## PO Confirmation, Logging & Daily Summary                                               |
| Structure Inventory Updates   | Set                       | Format inventory update payload                      | Calculate Dynamic Reorder Point              | Update Inventory Fields                         | ## Inventory Sync (Shopify)                                                                |
| Update Inventory Fields       | Shopify                   | Write updated inventory levels back to Shopify       | Structure Inventory Updates                   |                                                | ## Inventory Sync (Shopify)                                                                |
| Structure Analytics Data      | Set                       | Structure KPI analytics data                           | Check Reorder Point Reached                   | Write Dashboard Metrics, Write Scenario Planning, Sync to Accounting System | ## Analytics Outputs & System Sync                                                        |
| Write Dashboard Metrics       | Google Sheets             | Write KPI metrics to dashboard sheet                   | Structure Analytics Data                       |                                                | ## Analytics Outputs & System Sync                                                        |
| Write Scenario Planning       | Google Sheets             | Write scenario planning data                            | Structure Analytics Data                       |                                                | ## Analytics Outputs & System Sync                                                        |
| Sync to Accounting System     | HTTP Request              | Push financial KPIs to accounting API                   | Structure Analytics Data                       |                                                | ## Analytics Outputs & System Sync                                                        |
| Detect Slow-Movers            | Code                      | Identify slow-moving SKUs and suggest actions          | Calculate Sales Velocity                       | Structure Slow-Mover Data                       | ## Slow-Movers Detection & Actions                                                       |
| Structure Slow-Mover Data     | Set                       | Format slow-mover recommendations                       | Detect Slow-Movers                            | Alert Slow-Mover Suggestions                    | ## Slow-Movers Detection & Actions                                                       |
| Alert Slow-Mover Suggestions  | Slack                     | Send Slack alerts for slow movers                      | Structure Slow-Mover Data                      |                                                | ## Slow-Movers Detection & Actions                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Hourly Trigger Node:**  
   - Type: Schedule Trigger  
   - Configure to run every 1 hour.

2. **Create Workflow Configuration Node:**  
   - Type: Set  
   - Define parameters:  
     - shopifyStoreUrl: "https://your-store.myshopify.com"  
     - reorderPointMultiplier: 1.5  
     - safetyStockDays: 7  
     - budgetLimit: 50000  
     - largeOrderThreshold: 10000  
     - slowMoverThresholdDays: 90

3. **Create Shopify Data Fetch Nodes:**  
   - Get Inventory Levels: Shopify node, resource "inventoryLevel", authenticated with access token.  
   - Get Product Details: Shopify node, resource "product", operation "getAll", returnAll true.  
   - Get Last 30 Days Orders: Shopify node, resource "order", operation "getAll", returnAll true, filter createdAtMin to 30 days ago.

4. **Create Google Sheets Data Fetch Nodes:**  
   - Read Inventory Master: Google Sheets node, sheet name "Inventory Master", document ID from env var INVENTORY_MASTER_SHEET_ID, service account auth.  
   - Read Suppliers: Google Sheets node, sheet name "Suppliers", document ID from env var SUPPLIERS_SHEET_ID, service account auth.  
   - Read Purchase Order Log: Google Sheets node, sheet name "PO Log", document ID from env var PO_LOG_SHEET_ID, service account auth.

5. **Create Merge Node:**  
   - Merge mode "combine" with 6 inputs from all above data fetch nodes.

6. **Create Calculate Sales Velocity Node:**  
   - Code node with JavaScript parsing merged data to compute sales velocity metrics per SKU.

7. **Create Calculate Dynamic Reorder Point Node:**  
   - Code node running once per item calculating reorder points using avg_daily_sales, lead_time, safety_stock, and flags reorder need.

8. **Create Check Reorder Point Reached Node:**  
   - IF node checking if current stock <= reorder point.

9. **Create Calculate Stockout Risk Node:**  
   - Code node estimating stockout risk score (0-1 scale) based on days until stockout vs lead time.

10. **Create Check High Stockout Risk Node:**  
    - IF node for stockout risk > 0.7.

11. **Create Alert Critical Stock Risk Node:**  
    - Slack node sending alerts to configured Slack channel.

12. **Create Check Warehouse Redistribution Possible Node:**  
    - IF node checking presence of warehouse overstock and understock info.

13. **Create Multi-Warehouse Distribution Logic Node:**  
    - Code node analyzing warehouse inventory to recommend transfers.

14. **Create Structure Transfer Recommendations Node:**  
    - Set node formatting transfer recommendation data.

15. **Create Check Supplier Availability Node:**  
    - IF node checking supplier availability boolean.

16. **Create Enrich with Supplier Data Node:**  
    - Code node merging supplier info into SKU data.

17. **Create Check Business Day Node:**  
    - IF node allowing flow only on weekdays.

18. **Create Check MOQ Met Node:**  
    - IF node verifying order quantity >= MOQ.

19. **Create Check Promotional Period Node:**  
    - IF node filtering for promotional period flag.

20. **Create Check Budget Limit Node:**  
    - IF node checking total PO value < budget limit.

21. **Create Check Large Order Approval Needed Node:**  
    - IF node flagging large orders exceeding threshold.

22. **Create Calculate Order Quantity and Value Node:**  
    - Code node calculating adjusted order quantity (enforcing MOQ) and total PO value.

23. **Create Optimize Profit Priority Node:**  
    - Code node calculating profit score and sorting SKUs by priority.

24. **Create Structure PO Line Items Node:**  
    - Set node structuring PO line item fields with SKU, supplier, quantity, prices.

25. **Create Prepare PO Email Context Node:**  
    - Code node grouping PO line items by supplier, generating PO numbers, and formatting email HTML.

26. **Create Send PO Email Node:**  
    - Gmail node sending emails to supplier_email; authenticate with service account or OAuth2.

27. **Create Send PO to Supplier API Node:**  
    - HTTP Request node POSTing to supplier API endpoint with PO data, using API token from env var.

28. **Create Wait for PO Confirmation Node:**  
    - Wait node configured for 1 hour delay, listening for confirmation webhook.

29. **Create Update Purchase Order Log Node:**  
    - Google Sheets node appending/updating PO log with PO status, timestamps.

30. **Create Alert PO Sent Node:**  
    - Slack node notifying PO sent with details.

31. **Create Structure Inventory Updates Node:**  
    - Set node preparing inventory update payload for Shopify.

32. **Create Update Inventory Fields Node:**  
    - Shopify node updating inventory levels.

33. **Create Structure Analytics Data Node:**  
    - Set node structuring KPIs like turnover, carrying cost, stockout saves.

34. **Create Write Dashboard Metrics Node:**  
    - Google Sheets node writing KPI metrics to dashboard sheet.

35. **Create Write Scenario Planning Node:**  
    - Google Sheets node writing scenario planning data.

36. **Create Sync to Accounting System Node:**  
    - HTTP Request node POSTing analytics data to accounting API with HTTP header auth.

37. **Create Detect Slow-Movers Node:**  
    - Code node identifying slow-moving SKUs and generating discount/bundle suggestions.

38. **Create Structure Slow-Mover Data Node:**  
    - Set node formatting slow-mover data.

39. **Create Alert Slow-Mover Suggestions Node:**  
    - Slack node sending alerts on slow movers.

40. **Create Aggregate Daily Summary Node:**  
    - Code node aggregating daily PO counts, values, alerts, and slow movers safely.

41. **Create Send Daily Summary Node:**  
    - Slack node posting daily summary message.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                   | Context or Link                                                                                               |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| This workflow automates inventory monitoring, predictive reordering, and supplier communication using Shopify + Google Sheets. It runs hourly, merges multiple data sources, calculates sales velocity, reorder points, and risk scores, and automates PO creation and notifications. | Sticky Note (position: -3840,-224) "How it works" section                                                      |
| Setup includes connecting Shopify API credentials, Google Sheets IDs for Inventory Master, Suppliers, and PO Log, and Slack channels for alerts and summaries. Configure parameters like lead times, safety stock, budget limits, and approval thresholds for flexible operation.             | Sticky Note (position: -3840,-224) "Setup steps" section                                                       |
| Slack channels should be configured to receive critical stock alerts, slow-mover suggestions, PO notifications, and daily summaries.                                                                                                         | Environment variables for Slack channel IDs (e.g., SLACK_CRITICAL_CHANNEL)                                     |
| Google Sheets service accounts require proper permissions to read/write sheets named "Inventory Master", "Suppliers", "PO Log", "Metrics", and "Scenarios".                                                                                 | Environment variables for Google Sheets IDs (e.g., INVENTORY_MASTER_SHEET_ID)                                   |
| Shopify API rate limits and authentication must be managed carefully to avoid workflow disruptions.                                                                                                                                          | Shopify nodes using access tokens                                                                             |
| Supplier API endpoints and tokens should be secured and tested for PO submission integration.                                                                                                                                                 | Environment variables SUPPLIER_API_TOKEN, SUPPLIER_API_ENDPOINT                                               |
| The workflow uses multiple JavaScript Code nodes; ensure n8n supports version 2 code execution and handle exceptions in code for robustness.                                                                                                 | Node type: Code, Version 2                                                                                    |

---

**Disclaimer:** The provided workflow originates exclusively from an automated n8n workflow reflecting lawful and public data handling. It strictly respects content policies and contains no illegal or protected elements.
Automate Fundamental Stock Analysis with FinnHub Data and Google Sheets DCF Calculator

https://n8nworkflows.xyz/workflows/automate-fundamental-stock-analysis-with-finnhub-data-and-google-sheets-dcf-calculator-11674


# Automate Fundamental Stock Analysis with FinnHub Data and Google Sheets DCF Calculator

### 1. Workflow Overview

This workflow automates the fundamental stock analysis process by collecting financial data from FinnHub API and calculating a Discounted Cash Flow (DCF) valuation using a Google Sheets DCF calculator. It targets financial analysts or investors seeking automated, up-to-date fundamental analysis incorporating recent company filings, financial ratios, and market capitalization, ultimately producing structured outputs in Google Sheets for further review.

Logical blocks included:

- **1.1 Input Reception:** Receives trigger input specifying if the company’s most recent filing is a quarterly report.
- **1.2 Data Retrieval:** Fetches financial data including annual and quarterly reports, financial ratios, and market capitalization from FinnHub.
- **1.3 Data Aggregation and Calculation:** Aggregates raw data, calculates trailing twelve months (TTM) values, filters important financial metrics, and computes the DCF valuation.
- **1.4 Data Preparation for Output:** Prepares data structures for output into Google Sheets.
- **1.5 Google Sheets Operations:** Clears existing data and updates multiple sheets (GraphData, GrowthTable, DCFTable) with new analysis results.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
Starts the workflow with a form trigger to specify whether the most recent company filing is a quarterly report, which influences subsequent data fetching logic.

**Nodes Involved:**  
- Use if most recent company filing is quarterly report

**Node Details:**  

- **Use if most recent company filing is quarterly report**  
  - Type: Form Trigger  
  - Role: Entry point to receive user input or external event indicating filing type  
  - Configuration: Webhook triggered; no parameters configured within node, acts as start trigger  
  - Inputs: None (trigger node)  
  - Outputs: Four connections to four HTTP Request nodes (Annual Reports, Quarterly Reports, Ratios, MarketCap)  
  - Edge cases: Failure if webhook not reachable or if input is malformed/missing  

---

#### 2.2 Data Retrieval

**Overview:**  
Fetches financial data from FinnHub including annual and quarterly reports, financial ratios, and market capitalization.

**Nodes Involved:**  
- Annual Reports  
- Quarterly Reports  
- Ratios  
- MarketCap  

**Node Details:**  

- **Annual Reports**  
  - Type: HTTP Request  
  - Role: Retrieve annual financial filings from FinnHub API  
  - Configuration: Endpoint set to FinnHub filings API for annual reports, includes authentication credentials for FinnHub API  
  - Inputs: Trigger from form node  
  - Outputs: Merges with quarterly reports and calculated data downstream  
  - Edge cases: API rate limits, authentication failure, no annual data returned  

- **Quarterly Reports**  
  - Type: HTTP Request  
  - Role: Retrieve quarterly financial filings from FinnHub API  
  - Configuration: Endpoint set to FinnHub quarterly reports API, authenticated  
  - Inputs: Trigger from form node  
  - Outputs: Merges with annual reports and calculated data downstream  
  - Edge cases: API limits, missing quarterly data, malformed response  

- **Ratios**  
  - Type: HTTP Request  
  - Role: Obtains key financial ratios from FinnHub  
  - Configuration: FinnHub API endpoint for financial ratios, authenticated  
  - Inputs: Trigger from form node  
  - Outputs: Connected to aggregation node for combining with market cap data  
  - Edge cases: Data missing or inconsistent ratios, API failure  

- **MarketCap**  
  - Type: HTTP Request  
  - Role: Fetches market capitalization data from FinnHub  
  - Configuration: FinnHub API endpoint for market cap, authenticated  
  - Inputs: Trigger from form node  
  - Outputs: Connected to aggregation node with ratios  
  - Edge cases: API failure, delayed market data, authentication errors  

---

#### 2.3 Data Aggregation and Calculation

**Overview:**  
Aggregates fetched data, calculates trailing twelve months (TTM) financials, filters important financial metrics, and runs the DCF valuation calculation.

**Nodes Involved:**  
- Merge3  
- Aggregate1  
- Calculate TTM  
- Merge  
- Filter Important Financials  
- DCF Calculator  
- Merge1  

**Node Details:**  

- **Merge3**  
  - Type: Merge  
  - Role: Combines MarketCap and Ratios data streams  
  - Inputs: MarketCap (index 0), Ratios (index 1)  
  - Outputs: Aggregate1 node  
  - Edge cases: Missing one data stream causing incomplete merge  

- **Aggregate1**  
  - Type: Aggregate  
  - Role: Performs aggregation functions (sums, averages) on merged financial data  
  - Inputs: From Merge3  
  - Outputs: Calculate TTM  
  - Edge cases: Empty data sets causing aggregation errors  

- **Calculate TTM**  
  - Type: Code  
  - Role: Calculates trailing twelve months financial figures from aggregated data  
  - Inputs: Aggregate1 output  
  - Outputs: Merge node  
  - Key expressions: Custom JavaScript code computing TTM values, handling quarterly to annual conversion  
  - Edge cases: Missing quarterly data, division by zero, malformed input  

- **Merge**  
  - Type: Merge  
  - Role: Combines Calculate TTM output with Annual and Quarterly Reports  
  - Inputs: Calculate TTM (index 0), Annual Reports (index 1), Quarterly Reports (index 2)  
  - Outputs: Filter Important Financials  
  - Edge cases: Any input missing causes incomplete data set  

- **Filter Important Financials**  
  - Type: Code  
  - Role: Extracts key financial metrics relevant for analysis from combined data  
  - Inputs: Merge output  
  - Outputs: Two branches: DCF Calculator and Merge1  
  - Key expressions: JavaScript filters for specific financial ratios, earnings, cash flow data  
  - Edge cases: Key metrics missing, data format inconsistencies  

- **DCF Calculator**  
  - Type: Code  
  - Role: Calculates Discounted Cash Flow valuation based on filtered financial data  
  - Inputs: Filter Important Financials output  
  - Outputs: Merge1 (index 0)  
  - Key expressions: Custom JavaScript implementing DCF formula, discount rates, growth assumptions  
  - Edge cases: Invalid input numbers, division by zero, missing growth rate assumptions  

- **Merge1**  
  - Type: Merge  
  - Role: Combines DCF Calculator output with Filter Important Financials output (index 1)  
  - Inputs: DCF Calculator (index 0), Filter Important Financials (index 1)  
  - Outputs: Prep For Sheets  
  - Edge cases: Data inconsistency between branches  

---

#### 2.4 Data Preparation for Output

**Overview:**  
Prepares structured data for insertion into Google Sheets, splits data streams for different sheets.

**Nodes Involved:**  
- Prep For Sheets  
- Split Out  
- Split Out1  
- Split Out2  

**Node Details:**  

- **Prep For Sheets**  
  - Type: Code  
  - Role: Formats and structures data arrays and objects for Google Sheets update (GraphData, GrowthTable, DCFTable)  
  - Inputs: Merge1 output  
  - Outputs: Three Split Out nodes for parallel processing to different sheets  
  - Key expressions: JavaScript data mapping, array/object restructuring  
  - Edge cases: Missing data fields, incompatible data types for sheets  

- **Split Out**  
  - Type: Split Out  
  - Role: Extracts GraphData subset for Google Sheets  
  - Inputs: Prep For Sheets output  
  - Outputs: Clear GraphData  
  - Edge cases: Empty data sets  

- **Split Out1**  
  - Type: Split Out  
  - Role: Extracts GrowthTable subset for Google Sheets  
  - Inputs: Prep For Sheets output  
  - Outputs: Clear GrowthTable  
  - Edge cases: Empty data sets  

- **Split Out2**  
  - Type: Split Out  
  - Role: Extracts DCFTable subset for Google Sheets  
  - Inputs: Prep For Sheets output  
  - Outputs: Clear DCFTable  
  - Edge cases: Empty data sets  

---

#### 2.5 Google Sheets Operations

**Overview:**  
Clears existing data in target Google Sheets ranges and updates them with new analysis data.

**Nodes Involved:**  
- Clear GraphData  
- Update GraphData  
- Clear GrowthTable  
- Update Growth Table  
- Clear DCFTable  
- Update DCFTable  

**Node Details:**  

- **Clear GraphData**  
  - Type: Google Sheets  
  - Role: Clears previous data in the GraphData sheet/range  
  - Inputs: Split Out output  
  - Outputs: Update GraphData  
  - Configuration: Spreadsheet and sheet name set, clear values operation  
  - Edge cases: Authentication failure, sheet/range not found  

- **Update GraphData**  
  - Type: Google Sheets  
  - Role: Updates GraphData sheet with new financial graph data  
  - Inputs: Clear GraphData output  
  - Outputs: None (end of branch)  
  - Configuration: Spreadsheet and range configured, data values inserted  
  - Edge cases: Authentication, data size limits  

- **Clear GrowthTable**  
  - Type: Google Sheets  
  - Role: Clears previous growth table data  
  - Inputs: Split Out1 output  
  - Outputs: Update Growth Table  
  - Configuration: Spreadsheet and range set  
  - Edge cases: Invalid range, auth failures  

- **Update Growth Table**  
  - Type: Google Sheets  
  - Role: Inserts updated growth metrics data  
  - Inputs: Clear GrowthTable output  
  - Outputs: None (end of branch)  
  - Edge cases: Same as above  

- **Clear DCFTable**  
  - Type: Google Sheets  
  - Role: Clears previous DCF valuation data  
  - Inputs: Split Out2 output  
  - Outputs: Update DCFTable  
  - Configuration: Spreadsheet and range set  
  - Edge cases: Same as above  

- **Update DCFTable**  
  - Type: Google Sheets  
  - Role: Updates the DCF valuation table with new calculations  
  - Inputs: Clear DCFTable output  
  - Outputs: None (end of branch)  
  - Edge cases: Same as above  

---

### 3. Summary Table

| Node Name                                   | Node Type          | Functional Role                               | Input Node(s)                                     | Output Node(s)                                   | Sticky Note                                     |
|---------------------------------------------|--------------------|-----------------------------------------------|--------------------------------------------------|-------------------------------------------------|------------------------------------------------|
| Use if most recent company filing is quarterly report | Form Trigger       | Entry point, receive filing type input        | None                                             | Annual Reports, Quarterly Reports, Ratios, MarketCap |                                                |
| Annual Reports                              | HTTP Request       | Fetch annual financial reports                 | Use if most recent company filing is quarterly report | Merge                                          |                                                |
| Quarterly Reports                          | HTTP Request       | Fetch quarterly financial reports              | Use if most recent company filing is quarterly report | Merge                                          |                                                |
| Ratios                                    | HTTP Request       | Fetch financial ratios                          | Use if most recent company filing is quarterly report | Merge3                                         |                                                |
| MarketCap                                 | HTTP Request       | Fetch market capitalization data               | Use if most recent company filing is quarterly report | Merge3                                         |                                                |
| Merge3                                    | Merge              | Combine MarketCap and Ratios                    | MarketCap, Ratios                                 | Aggregate1                                      |                                                |
| Aggregate1                                | Aggregate          | Aggregate combined financial data               | Merge3                                            | Calculate TTM                                   |                                                |
| Calculate TTM                             | Code               | Calculate trailing twelve months financials    | Aggregate1                                        | Merge                                           |                                                |
| Merge                                     | Merge              | Combine TTM, Annual, and Quarterly reports      | Calculate TTM, Annual Reports, Quarterly Reports | Filter Important Financials                      |                                                |
| Filter Important Financials               | Code               | Filter key financial metrics                     | Merge                                             | DCF Calculator, Merge1                          |                                                |
| DCF Calculator                           | Code               | Calculate DCF valuation                           | Filter Important Financials                       | Merge1                                           |                                                |
| Merge1                                    | Merge              | Combine DCF results and filtered financials     | DCF Calculator, Filter Important Financials      | Prep For Sheets                                 |                                                |
| Prep For Sheets                          | Code               | Prepare data for Google Sheets                    | Merge1                                            | Split Out, Split Out1, Split Out2                |                                                |
| Split Out                                | Split Out          | Extract GraphData subset                           | Prep For Sheets                                   | Clear GraphData                                 |                                                |
| Split Out1                               | Split Out          | Extract GrowthTable subset                         | Prep For Sheets                                   | Clear GrowthTable                               |                                                |
| Split Out2                               | Split Out          | Extract DCFTable subset                            | Prep For Sheets                                   | Clear DCFTable                                  |                                                |
| Clear GraphData                          | Google Sheets      | Clear previous GraphData                           | Split Out                                         | Update GraphData                                |                                                |
| Update GraphData                        | Google Sheets      | Update GraphData with new data                      | Clear GraphData                                   | None                                            |                                                |
| Clear GrowthTable                       | Google Sheets      | Clear previous GrowthTable data                     | Split Out1                                        | Update Growth Table                             |                                                |
| Update Growth Table                    | Google Sheets      | Update GrowthTable with new data                      | Clear GrowthTable                                 | None                                            |                                                |
| Clear DCFTable                         | Google Sheets      | Clear previous DCFTable data                         | Split Out2                                        | Update DCFTable                                 |                                                |
| Update DCFTable                      | Google Sheets      | Update DCFTable with new DCF calculations             | Clear DCFTable                                    | None                                            |                                                |
| Sticky Note                            | Sticky Note        | (Empty content)                                     | None                                             | None                                            |                                                |
| Sticky Note2                           | Sticky Note        | (Empty content)                                     | None                                             | None                                            |                                                |
| Sticky Note4                           | Sticky Note        | (Empty content)                                     | None                                             | None                                            |                                                |
| Sticky Note5                           | Sticky Note        | (Empty content)                                     | None                                             | None                                            |                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Form Trigger Node**  
   - Type: Form Trigger  
   - Name: "Use if most recent company filing is quarterly report"  
   - Configuration: Default webhook, no parameters needed  
   - Position: Start node  

2. **Create HTTP Request Nodes for Data Retrieval**  
   - Annual Reports: HTTP Request node  
     - Configure FinnHub API endpoint for annual filings  
     - Add FinnHub API credentials  
     - Connect input from Form Trigger  
   - Quarterly Reports: HTTP Request node  
     - Configure FinnHub API endpoint for quarterly filings  
     - Add FinnHub API credentials  
     - Connect input from Form Trigger  
   - Ratios: HTTP Request node  
     - Configure FinnHub API endpoint for financial ratios  
     - Add FinnHub API credentials  
     - Connect input from Form Trigger  
   - MarketCap: HTTP Request node  
     - Configure FinnHub API endpoint for market capitalization  
     - Add FinnHub API credentials  
     - Connect input from Form Trigger  

3. **Merge MarketCap and Ratios**  
   - Add a Merge node named "Merge3"  
   - Set to Merge by Index or Wait for all inputs  
   - Connect MarketCap (index 0) and Ratios (index 1)  

4. **Aggregate Financial Data**  
   - Add an Aggregate node named "Aggregate1"  
   - Configure aggregation rules as needed (sum, average) on incoming data  
   - Connect input from Merge3  

5. **Calculate TTM**  
   - Add a Code node named "Calculate TTM"  
   - Write JavaScript to calculate trailing twelve months from aggregated data  
   - Connect input from Aggregate1  

6. **Merge TTM, Annual Reports, Quarterly Reports**  
   - Add a Merge node named "Merge"  
   - Connect Calculate TTM (index 0), Annual Reports (index 1), Quarterly Reports (index 2)  

7. **Filter Important Financials**  
   - Add a Code node named "Filter Important Financials"  
   - Implement JavaScript to filter key metrics such as earnings, cash flow, ratios  
   - Connect input from Merge  

8. **Calculate DCF Valuation**  
   - Add a Code node named "DCF Calculator"  
   - Implement DCF formula calculating present value of future cash flows  
   - Connect input from Filter Important Financials  

9. **Merge DCF and Filtered Financials**  
   - Add a Merge node named "Merge1"  
   - Connect DCF Calculator output (index 0) and Filter Important Financials output (index 1)  

10. **Prepare Data for Google Sheets**  
    - Add a Code node named "Prep For Sheets"  
    - Format data into three distinct datasets: GraphData, GrowthTable, DCFTable  
    - Connect input from Merge1  

11. **Split Data Streams**  
    - Add three Split Out nodes named "Split Out", "Split Out1", "Split Out2"  
    - Connect all from Prep For Sheets node  
    - Each Split Out extracts one dataset for respective sheet  

12. **Google Sheets Operations per Dataset**  
    For each dataset:

    - Add a Google Sheets node to clear previous data (e.g., "Clear GraphData")  
      - Configure spreadsheet ID, sheet name, and range  
      - Set operation to clear or delete values  
      - Connect from corresponding Split Out node  

    - Add a Google Sheets node to update the sheet (e.g., "Update GraphData")  
      - Configure same spreadsheet and sheet name  
      - Set append or update values with prepared data  
      - Connect from respective clear node  

    Repeat for GrowthTable and DCFTable datasets with respective nodes.

13. **Credential Setup**  
    - Configure FinnHub API credentials (token-based) in n8n credentials manager  
    - Configure Google Sheets OAuth2 credentials with access to target spreadsheets  

14. **Testing and Validation**  
    - Test webhook trigger with sample input indicating filing type  
    - Validate API calls receive proper data and handle empty or error responses gracefully  
    - Verify that Google Sheets are updated correctly with new data  

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                             |
|-----------------------------------------------------------------------------------------------------------|--------------------------------------------|
| The workflow requires FinnHub API credentials with sufficient quota to avoid rate limiting during data fetch. | FinnHub API: https://finnhub.io/docs/api  |
| Google Sheets credentials must have editor access to the target spreadsheet and the relevant sheets must exist. | Google Sheets API: https://developers.google.com/sheets/api |
| The DCF Calculator code block implements financial modeling assumptions which can be customized as needed. | Customize discount rate, growth rate, and cash flow projections in the code node. |
| The workflow assumes consistent and complete financial data availability from FinnHub; missing data can cause calculation errors. | Consider adding error handling or data validation nodes for production use. |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and publicly available.
Import Products from Google Sheets to Shopify with Multi-Variant Support

https://n8nworkflows.xyz/workflows/import-products-from-google-sheets-to-shopify-with-multi-variant-support-9833


# Import Products from Google Sheets to Shopify with Multi-Variant Support

### 1. Workflow Overview

This workflow automates the import of product data from a Google Sheets spreadsheet into a Shopify store, supporting both single products and products with multiple variants (e.g., different sizes). It is designed for Shopify store owners or managers who maintain product inventories in spreadsheets and want to efficiently upload and synchronize that data with Shopify’s product catalog, including variant details and inventory quantities.

The workflow is logically divided into these blocks:

- **1.1 Input Reception and Preparation**: Triggering the workflow manually and retrieving product rows from Google Sheets.
- **1.2 Shopify Store Setup**: Setting Shopify store API endpoint and retrieving store location data needed for inventory management.
- **1.3 Product Data Processing and Classification**: Processing raw spreadsheet data to classify products as single or variant products and structuring variant data accordingly.
- **1.4 Conditional Branching**: Routing data flow depending on whether the product is a variant or a single product.
- **1.5 Shopify Product Creation**: Creating products in Shopify via GraphQL mutations; separate flows for variant and single products.
- **1.6 Variant Adjustment and Bulk Creation**: For variant products, adjusting variant options and creating/updating variants in bulk.
- **1.7 Inventory Management**: Setting inventory quantities on Shopify for each created variant or single product.
- **1.8 Utility and Configuration Nodes**: Nodes for setting URL, managing data transformations, and explanatory sticky notes.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Preparation

- **Overview:** This block initiates the workflow manually and fetches product data from a specific Google Sheets document and sheet.
- **Nodes Involved:**  
  - `When clicking ‘Execute workflow’`  
  - `set shop url`  
  - `Shopify, GetLocations`  
  - `Get row(s) in sheet`
- **Node Details:**  
  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Starts the workflow on manual execution  
    - Config: No parameters; triggers downstream nodes  
    - Output: Triggers `set shop url`  
    - Potential issues: None (manual trigger)  
  - **set shop url**  
    - Type: Set  
    - Role: Defines Shopify GraphQL API endpoint URL with placeholder for user’s store domain  
    - Config: Sets a string variable `myshop` with the URL `https://[yourshop].myshopify.com/admin/api/2025-04/graphql.json`  
    - Output: Triggers `Shopify, GetLocations`  
    - Edge cases: User must replace `[yourshop]` with actual store subdomain; otherwise, API calls fail. Sticky note reminds user about this.  
  - **Shopify, GetLocations**  
    - Type: GraphQL  
    - Role: Queries Shopify store locations (needed for inventory updates)  
    - Config: GraphQL query to fetch the first location’s id and address details  
    - Endpoint: Uses `myshop` URL from `set shop url` node  
    - Authentication: Header Auth with Shopify Admin API token  
    - Executes once per workflow run  
    - Output: Triggers `Get row(s) in sheet`  
    - Failure modes: Auth errors if token invalid, network timeouts, API rate limits  
  - **Get row(s) in sheet**  
    - Type: Google Sheets  
    - Role: Reads product data rows from the specified Google Sheet and sheet tab named "Products"  
    - Config: Spreadsheet ID and sheet tab ID are preset; credentials use OAuth2 Google Sheets account  
    - Output: Passes data to `single and multivariant products` node  
    - Edge cases: Credential expiry, sheet access rights, empty or malformed rows

#### 1.2 Product Data Processing and Classification

- **Overview:** Processes spreadsheet rows to group products by name, detect variant status, and prepare structured variant data for Shopify API.  
- **Nodes Involved:**  
  - `single and multivariant products`
- **Node Details:**  
  - **single and multivariant products**  
    - Type: Code (JavaScript)  
    - Role: Parses product rows; groups items by product name; identifies if products have multiple variants or are single; builds Shopify-compatible variant option structures and metadata.  
    - Key Logic:  
      - SKU parsing to identify base SKU and variant suffix  
      - Groups products by name  
      - Detects variants by count or size attribute  
      - Constructs `optionsGraph` for Shopify product options (e.g., Size)  
      - Builds arrays for variant creation and inventory updates  
    - Outputs objects with `type` field as either `"variant"` or `"single"` for downstream branching  
    - Edge cases: Improper SKU formatting, missing size data, inconsistent inventory numbers  
    - Failure: Code errors if expected fields missing, but none explicitly handled

#### 1.3 Conditional Branching

- **Overview:** Routes product data into separate flows for variant and single products.  
- **Nodes Involved:**  
  - `is variant?`
- **Node Details:**  
  - **is variant?**  
    - Type: Switch  
    - Role: Checks `type` property in JSON to route items either to variant or single product creation nodes  
    - Conditions:  
      - Output "Variant" if `$json.type === "variant"`  
      - Output "Single" if `$json.type === "single"`  
    - Outputs:  
      - Variant branch → `Shopify, CreateProduct`  
      - Single branch → `CreateProduct2`  
    - Edge cases: Unexpected or missing type field leads to no output path — may stall workflow

#### 1.4 Shopify Product Creation

- **Overview:** Creates new products in Shopify using GraphQL mutations; separate nodes handle variant and single product creation.  
- **Nodes Involved:**  
  - `Shopify, CreateProduct` (for variants)  
  - `CreateProduct2` (for single products)
- **Node Details:**  
  - **Shopify, CreateProduct**  
    - Type: GraphQL  
    - Role: Creates product with multiple variants and media  
    - Config: GraphQL mutation `productCreate` with variables including product title, vendor, type, handle, product options (variants), and media image  
    - Endpoint: Shopify API URL from `set shop url`  
    - Authentication: Header Auth with Shopify token  
    - Outputs: Product data including assigned IDs for variants and media  
    - Edge cases: API validation errors, network failures, missing required fields  
  - **CreateProduct2**  
    - Type: GraphQL  
    - Role: Creates single-variant product with media  
    - Config: Similar mutation but handles single product without variant options  
    - Outputs: Product and variant data for single product  
    - Edge cases: Same as above

#### 1.5 Variant Adjustment and Bulk Variant Creation

- **Overview:** Processes created variant products to adjust variant options, create bulk variants, and update variant details.  
- **Nodes Involved:**  
  - `adjust variants` (code node)  
  - `Split Out1` (Split Out node)  
  - `SetVariant` (GraphQL mutation for bulk variant creation)  
  - `set variants data` (Set node)  
  - `Update Variants` (GraphQL mutation for bulk variant updates)
- **Node Details:**  
  - **adjust variants**  
    - Type: Code  
    - Role: Matches original variant data to created product variants; builds bulk variant creation payloads with matched optionValue IDs and pricing  
    - Input: Created product data and original variant input data  
    - Output: JSON objects including `productId`, `mainfirstVariant`, variant lists with proper option IDs  
    - Failure risks: If Shopify product options do not contain expected "Size" option, throws error  
  - **Split Out1**  
    - Type: Split Out  
    - Role: Splits array of variant data into individual items for processing  
    - Output: Each variant item separately for mutation nodes  
  - **SetVariant**  
    - Type: GraphQL  
    - Role: Bulk creates variants for a product using Shopify mutation `productVariantsBulkCreate`  
    - Variables: Uses `productId` and array of variant objects with pricing and option IDs  
    - Output: Created variant data for further updates  
    - Edge cases: API errors, validation errors on variant data  
  - **set variants data**  
    - Type: Set  
    - Role: Prepares data variables for the next update mutation, extracting variant IDs and other details from previous node outputs  
  - **Update Variants**  
    - Type: GraphQL  
    - Role: Updates variant pricing and inventory tracking settings using `productVariantsBulkUpdate` mutation  
    - Input: Uses variant IDs and pricing info  
    - Output: Updated variant data to be used in inventory setting

#### 1.6 Inventory Management

- **Overview:** Sets inventory quantities for each created product variant or single product in Shopify at the store location retrieved earlier.  
- **Nodes Involved:**  
  - `SetInventory` (for variants)  
  - `Create SetInventory` (for single products)
- **Node Details:**  
  - **SetInventory**  
    - Type: GraphQL  
    - Role: Sets on-hand inventory quantities for variant inventory items using `inventorySetOnHandQuantities` mutation  
    - Uses inventory item IDs from updated variant data and location ID from `Shopify, GetLocations`  
    - Edge cases: Inventory quantity mismatch or Shopify API errors  
  - **Create SetInventory**  
    - Type: GraphQL  
    - Role: Same as above but for single product variant inventory  
    - Uses variant inventory item ID from single product creation node

#### 1.7 Utility and Configuration Nodes

- **Overview:** Supporting nodes for setting variables, managing data flow, and providing user instructions.  
- **Nodes Involved:**  
  - `set shop url` (discussed above)  
  - `Sticky Note` nodes (various)  
- **Node Details:**  
  - Sticky Notes provide setup instructions, branding, and usage tips:  
    - Instructions on Shopify credential creation and setup  
    - Reminder to replace placeholder shop URL  
    - Notes on variant splitting logic  
    - Recommendations on vendor and product type customization  
  - These do not affect data flow but are critical for user understanding and setup correctness.

---

### 3. Summary Table

| Node Name                     | Node Type          | Functional Role                                             | Input Node(s)                  | Output Node(s)                   | Sticky Note                                                                                                                          |
|-------------------------------|--------------------|-------------------------------------------------------------|-------------------------------|---------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger     | Starts workflow manually                                    | —                             | set shop url                    |                                                                                                                                      |
| set shop url                  | Set                | Sets Shopify API endpoint URL                               | When clicking ‘Execute workflow’ | Shopify, GetLocations           | ## Add your store subdomain \n**https://[yourshop].myshopify.com/**                                                                 |
| Shopify, GetLocations         | GraphQL            | Retrieves Shopify store location ID                         | set shop url                  | Get row(s) in sheet             |                                                                                                                                      |
| Get row(s) in sheet           | Google Sheets      | Fetches product rows from Google Sheets                     | Shopify, GetLocations         | single and multivariant products |                                                                                                                                      |
| single and multivariant products | Code              | Processes rows; groups products; detects variants           | Get row(s) in sheet           | is variant?                    |                                                                                                                                      |
| is variant?                  | Switch             | Routes products to variant or single product flows          | single and multivariant products | Shopify, CreateProduct (variant branch), CreateProduct2 (single branch) |                                                                                                                                      |
| Shopify, CreateProduct        | GraphQL            | Creates variant product in Shopify                          | is variant?                   | adjust variants                | ### Use fixed 'type' and 'vendor' or update the json variables to pass these variables.                                               |
| adjust variants               | Code               | Builds bulk variant creation payloads for Shopify          | Shopify, CreateProduct        | Split Out1                    | ### this  split node will split variants into individual items to be created and updated in next steps                              |
| Split Out1                   | Split Out          | Splits variant arrays into individual items                 | adjust variants               | SetVariant                    | ### this  split node will split variants into individual items to be created and updated in next steps                              |
| SetVariant                   | GraphQL            | Bulk creates product variants in Shopify                    | Split Out1                   | set variants data             | ### this  split node will split variants into individual items to be created and updated in next steps                              |
| set variants data            | Set                | Prepares variant data for update mutations                  | SetVariant                   | Update Variants               |                                                                                                                                      |
| Update Variants              | GraphQL            | Updates variant pricing and inventory tracking              | set variants data            | SetInventory                  |                                                                                                                                      |
| SetInventory                 | GraphQL            | Sets inventory quantities for variants                      | Update Variants              | —                            |                                                                                                                                      |
| CreateProduct2               | GraphQL            | Creates single product in Shopify                            | is variant?                  | SetVariant1                  |                                                                                                                                      |
| SetVariant1                  | GraphQL            | Updates single product variant price and tracking           | CreateProduct2               | Create SetInventory           |                                                                                                                                      |
| Create SetInventory          | GraphQL            | Sets inventory quantity for single product variant          | SetVariant1                  | —                            |                                                                                                                                      |
| Sticky Note                  | Sticky Note        | Workflow branding and setup instructions                    | —                             | —                            | ![Shopify Logo](https://upload.wikimedia.org/wikipedia/commons/thumb/0/0e/Shopify_logo_2018.svg/1280px-Shopify_logo_2018.svg.png)\n# Shopify Product Importer with Variants Support\n\n## Setup Guidelines\n\n🔧 Setup Instructions\n### Step 1: Configure Shopify Credentials\nIn n8n, create a new Header Auth credential\nSet these values:\n\nName: X-Shopify-Access-Token\nValue: Your Shopify Admin API Access Token\n\n### Step 2: Update Workflow Nodes\nShopify, GetLocations node:\nUpdate the endpoint with your store URL\nSelect your Shopify credentials\n\nGet row(s) in sheet node:\nConnect your Google Sheets account\nSelect your spreadsheet\nChoose the sheet with product data\n\nUpdate all other Shopify nodes with:\nYour store URL (replace myshop.myshopify.com)\nYour Shopify credentials\n\n### Step 3: Customize Product Details (Optional)\nFor variant products (in Shopify, CreateProduct node):\n\"vendor\": \"vendor\",  // Change to your vendor name\n\"productType\": \"type\",  // Change to your product type\n\nFor single products (in CreateProduct2 node):\n\"vendor\": \"vendor\",  // Change to your vendor name\n\"productType\": \"type\",  // Change to your product type |
| Sticky Note1                 | Sticky Note        | Reminds user to add store subdomain in URL                 | —                             | —                            | ## Add your store subdomain \n**https://[yourshop].myshopify.com/**                                                                  |
| Sticky Note2                 | Sticky Note        | Notes on using fixed 'type' and 'vendor' in JSON variables | —                             | —                            | ### Use fixed 'type' and 'vendor' or update the json variables to pass these variables.                                               |
| Sticky Note3                 | Sticky Note        | Explains purpose of split node for variants                 | —                             | —                            | ### this  split node will split variants into individual items to be created and updated in next steps                              |
| Sticky Note4                 | Sticky Note        | Explains the table source for products                      | —                             | —                            | ## Connect your table ( source of products)\n** table with title, SKU, picture url, price, inventory and/or vendor, type **           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Add a Manual Trigger node named `When clicking ‘Execute workflow’`. No parameters needed.

2. **Create Set Node (`set shop url`):**  
   - Add a Set node named `set shop url`.  
   - Add a string field named `myshop`.  
   - Set its value to your Shopify API GraphQL endpoint URL:  
     `https://[yourshop].myshopify.com/admin/api/2025-04/graphql.json` replacing `[yourshop]` with your store subdomain.

3. **Create Shopify GetLocations Node:**  
   - Add a GraphQL node named `Shopify, GetLocations`.  
   - Set the endpoint to `={{ $('set shop url').item.json.myshop }}`.  
   - Enter this query:

     ```
     query {
         locations(first:1, reverse:true) {
             edges {
                 node {
                     id
                     name
                     address {
                         address1
                         address2
                         city
                         country
                         zip
                         province
                     }
                 }
             }
         }
     }
     ```
   - Authentication: Use Header Auth credentials with your Shopify Admin API Access Token.  
   - Set `Execute Once` option to true.

4. **Create Google Sheets Node:**  
   - Add a Google Sheets node named `Get row(s) in sheet`.  
   - Connect your Google Sheets OAuth2 credentials.  
   - Set the Spreadsheet ID to your product data spreadsheet.  
   - Set the Sheet Name to `"Products"` (or your actual sheet tab).  
   - Leave options default.

5. **Create Code Node (`single and multivariant products`):**  
   - Add a Code node named `single and multivariant products`.  
   - Use the provided JavaScript code that:  
     - Groups products by name  
     - Parses SKUs for variants  
     - Builds Shopify optionsGraph and variant arrays  
     - Determines product type (`variant` or `single`)  
   - No input parameters; code uses input items from Sheets node.

6. **Create Switch Node (`is variant?`):**  
   - Add a Switch node named `is variant?`.  
   - Add two outputs:  
     - Output "Variant" for items with `$json.type === "variant"`.  
     - Output "Single" for items with `$json.type === "single"`.

7. **Create Shopify Product Creation Nodes:**  
   - **Variant products:**  
     - Add a GraphQL node named `Shopify, CreateProduct`.  
     - Endpoint: `={{ $('set shop url').first().json.myshop }}`.  
     - Use mutation `productCreate` with variables referencing:  
       - `productName`, `optionsGraph`, and `productImage` from input JSON.  
     - Authenticate with Shopify Header Auth credentials.  
   - **Single products:**  
     - Add another GraphQL node named `CreateProduct2`.  
     - Similar setup as above but mutation suited for single variant product.

8. **Create Code Node (`adjust variants`):**  
   - Add a Code node named `adjust variants`.  
   - This node takes the created product data and original variant data and builds payloads for bulk variant creation.  
   - Includes error handling for missing 'Size' option.

9. **Create Split Out Node (`Split Out1`):**  
   - Add a Split Out node named `Split Out1`.  
   - Configure to split the `variants` field into individual output items.  
   - Include other fields: `productId`, `productTitle`, `mainfirstVariant`, `mediaID`.

10. **Create GraphQL Node (`SetVariant`):**  
    - Add a GraphQL node named `SetVariant`.  
    - Mutation `productVariantsBulkCreate` with variables `productId` and `variants` array.  
    - Use data from `Split Out1` items.  
    - Authenticate with Shopify credentials.

11. **Create Set Node (`set variants data`):**  
    - Add a Set node named `set variants data`.  
    - Map variant and product IDs from previous nodes to variables used for updates.

12. **Create GraphQL Node (`Update Variants`):**  
    - Add a GraphQL node named `Update Variants`.  
    - Mutation `productVariantsBulkUpdate` to update variant pricing and inventory tracking.  
    - Use variables mapped in `set variants data`.

13. **Create GraphQL Node (`SetInventory`):**  
    - Add a GraphQL node named `SetInventory`.  
    - Mutation `inventorySetOnHandQuantities` to set inventory quantities for variants.  
    - Use inventory item IDs and location ID from `Shopify, GetLocations`.

14. **Create GraphQL Node (`SetVariant1`) for Single Product Variant Update:**  
    - Add a GraphQL node named `SetVariant1`.  
    - Mutation `productVariantsBulkUpdate` for the single product variant.  
    - Use product ID and variant ID from `CreateProduct2`.

15. **Create GraphQL Node (`Create SetInventory`) for Single Product Inventory:**  
    - Add a GraphQL node named `Create SetInventory`.  
    - Same mutation as `SetInventory` but for single product variant.

16. **Link Nodes According to the workflow:**  
    - `When clicking ‘Execute workflow’` → `set shop url` → `Shopify, GetLocations` → `Get row(s) in sheet` → `single and multivariant products` → `is variant?`  
    - `is variant?` Variant output → `Shopify, CreateProduct` → `adjust variants` → `Split Out1` → `SetVariant` → `set variants data` → `Update Variants` → `SetInventory`  
    - `is variant?` Single output → `CreateProduct2` → `SetVariant1` → `Create SetInventory`

17. **Set Credentials:**  
    - Create and configure Google Sheets OAuth2 credentials.  
    - Create Shopify Header Auth credentials with:  
      - Name: `X-Shopify-Access-Token`  
      - Value: Your Shopify Admin API Access Token.

18. **Customize Product Details:**  
    - Edit GraphQL mutation variables for vendor and product type if desired.  
    - Replace placeholder text in `set shop url`.

19. **Add Sticky Notes (Optional):**  
    - Add notes with instructions, branding, and reminders for user clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                   | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| ![Shopify Logo](https://upload.wikimedia.org/wikipedia/commons/thumb/0/0e/Shopify_logo_2018.svg/1280px-Shopify_logo_2018.svg.png)\n\nShopify Product Importer with Variants Support\n\nSetup instructions for Shopify credentials and Google Sheets. | Main workflow branding and setup instructions                                                  |
| Add your store subdomain in the Shopify API endpoint URL, replacing `[yourshop]`. Example: `https://mystore.myshopify.com/admin/api/2025-04/graphql.json`                                                                                       | Sticky Note1 content                                                                              |
| Use fixed 'type' and 'vendor' values in GraphQL mutation variables or modify JSON variables as needed to customize product details.                                                                                                           | Sticky Note2 content                                                                              |
| The Split Out node splits variant arrays into individual items for creation and update steps, enabling bulk operations on variants.                                                                                                           | Sticky Note3 content                                                                              |
| Connect your Google Sheets table containing product data with columns for title, SKU, picture URL, price, inventory, vendor, and type.                                                                                                        | Sticky Note4 content                                                                              |

---

This document provides a complete technical reference to understand, reproduce, and modify the “Import Products from Google Sheets to Shopify with Multi-Variant Support” n8n workflow. It highlights the logical structure, node configurations, data flows, and common failure modes, ensuring maintainers and developers can maintain or extend the automation reliably.
Sync Products Between Airtable and Shopify with Inventory Management

https://n8nworkflows.xyz/workflows/sync-products-between-airtable-and-shopify-with-inventory-management-7468


# Sync Products Between Airtable and Shopify with Inventory Management

### 1. Workflow Overview

This workflow synchronizes product data from Airtable into a Shopify store, creating or updating Shopify products along with their inventory information. It is designed for use cases where Airtable acts as the primary product data source, and Shopify needs to reflect those products accurately, including inventory quantity management at the default store location.

The workflow’s logical structure is organized into these blocks:

- **1.1 Initialization and Location Retrieval:** Starts the workflow and fetches Shopify store locations needed for inventory assignment.
- **1.2 Airtable Data Fetch:** Retrieves products marked for synchronization from Airtable.
- **1.3 Product Loop and Shopify Product Query:** Processes each Airtable product record individually, querying Shopify by product handle (slug) to determine if the product exists.
- **1.4 Product Existence Decision:** Branches logic based on whether the Shopify product exists or not.
- **1.5 Shopify Product Update:** Updates existing Shopify products and their variants, then sets inventory quantities.
- **1.6 Shopify Product Creation:** Creates new Shopify products and their variants, then sets inventory quantities.
- **1.7 Airtable Sync Flag Update:** Marks Airtable records as synchronized after successful Shopify operations.
- **1.8 Workflow Completion:** Ends the workflow cycle.

---

### 2. Block-by-Block Analysis

#### 1.1 Initialization and Location Retrieval

- **Overview:** Triggered manually, this block starts the workflow and obtains the Shopify store’s location ID for inventory operations.
- **Nodes Involved:** Start Workflow, Shopify, GetLocations
- **Node Details:**
  - **Start Workflow**
    - Type: Manual Trigger
    - Role: Entry point for manual execution.
    - Configuration: No parameters; triggers on demand.
    - Inputs: None
    - Outputs: Shopify, GetLocations
    - Failure Modes: None typical; manual trigger.
  - **Shopify, GetLocations**
    - Type: GraphQL Query
    - Role: Fetches Shopify store locations (only the first location, reversed order).
    - Configuration: GraphQL query for locations with ID and address details.
    - Key Variables: None dynamic; static query.
    - Inputs: From Start Workflow
    - Outputs: Airtable, FetchRecords
    - Credentials: Shopify GraphQL Header Authentication
    - Failure Modes: API rate limiting, auth errors, network issues.
    - Notes: Executes once per workflow run.

#### 1.2 Airtable Data Fetch

- **Overview:** Retrieves Airtable product records flagged for synchronization (`sync` column set to true).
- **Nodes Involved:** Airtable, FetchRecords
- **Node Details:**
  - Type: Airtable node
  - Role: Search operation for product records to sync.
  - Configuration: Base and table selected; fields include title, description, company, status, slug, price, etc.
  - Filter: Only records where `sync` is true.
  - Inputs: Shopify, GetLocations
  - Outputs: Loop
  - Credentials: Airtable Personal Access Token
  - Failure Modes: API key invalid, rate limits, formula errors.

#### 1.3 Product Loop and Shopify Product Query

- **Overview:** Processes each Airtable product record individually; queries Shopify by product handle (slug) to check for existence.
- **Nodes Involved:** Loop, Shopify, ProductQuery
- **Node Details:**
  - **Loop**
    - Type: SplitInBatches
    - Role: Iterates over Airtable records one at a time.
    - Configuration: Default batch size (1).
    - Inputs: Airtable, FetchRecords
    - Outputs: Shopify, ProductQuery and Finished
    - Failure Modes: Empty input arrays, batch processing errors.
  - **Shopify, ProductQuery**
    - Type: GraphQL Query
    - Role: Queries Shopify product by handle (slug).
    - Configuration: GraphQL productByHandle query with variables set dynamically from current Loop item’s slug.
    - Inputs: Loop
    - Outputs: If product exists
    - Credentials: Shopify GraphQL Header Authentication
    - Failure Modes: Auth errors, non-existent handle returns null, query syntax errors.

#### 1.4 Product Existence Decision

- **Overview:** Decides workflow path based on whether the Shopify product exists.
- **Nodes Involved:** If product exists
- **Node Details:**
  - Type: If node
  - Role: Checks if `data.productByHandle` exists in Shopify query response.
  - Configuration: Condition is an existence check on `{{$json.data.productByHandle}}`.
  - Inputs: Shopify, ProductQuery
  - Outputs: Shopify, UpdateProduct (true branch), Shopify, CreateProduct (false branch)
  - Failure Modes: Expression syntax errors, unexpected nulls.

#### 1.5 Shopify Product Update Path

- **Overview:** Updates the existing Shopify product details, bulk updates variants, sets inventory quantity, and marks Airtable record synced.
- **Nodes Involved:** Shopify, UpdateProduct; Shopify, Update SetVariant; Shopify, Update SetInventory; Airtable, MarkRecord as Sync'd
- **Node Details:**
  - **Shopify, UpdateProduct**
    - Type: GraphQL Mutation
    - Role: Updates product metadata (title, description, vendor, type, status, handle) and media.
    - Configuration: Variables dynamically sourced from Loop item and Shopify product ID.
    - Inputs: If product exists (true branch)
    - Outputs: Shopify, Update SetVariant
    - Failure Modes: User errors from Shopify, auth failures.
  - **Shopify, Update SetVariant**
    - Type: GraphQL Mutation
    - Role: Bulk update product variant details (price, compare-at-price, SKU, inventory tracking).
    - Configuration: Uses product ID and variant ID from update response, variant details from Loop item.
    - Inputs: Shopify, UpdateProduct
    - Outputs: Shopify, Update SetInventory
    - Failure Modes: Variant mismatch, mutation failures.
  - **Shopify, Update SetInventory**
    - Type: GraphQL Mutation
    - Role: Sets inventory quantity for the updated variant at the store location.
    - Configuration: Inventory item ID from variant, location ID from initial location fetch, quantity from Loop.
    - Inputs: Shopify, Update SetVariant
    - Outputs: Airtable, MarkRecord as Sync'd
    - Failure Modes: Inventory item not found, permissions errors.
  - **Airtable, MarkRecord as Sync'd**
    - Type: Airtable Update
    - Role: Marks the Airtable record’s `sync` field as false (indicating sync complete).
    - Configuration: Matches record by Airtable ID from Loop item.
    - Inputs: Shopify, Update SetInventory
    - Outputs: Loop (to process next record)
    - Failure Modes: Airtable API errors, record ID mismatch.

#### 1.6 Shopify Product Creation Path

- **Overview:** Creates new Shopify products and variants, sets inventory, then updates Airtable sync flag.
- **Nodes Involved:** Shopify, CreateProduct; Shopify, Create SetVariant; Shopify, Create SetInventory; Airtable, MarkRecord as Sync'd
- **Node Details:**
  - **Shopify, CreateProduct**
    - Type: GraphQL Mutation
    - Role: Creates a new product with metadata and media.
    - Configuration: Variables sourced from Loop item.
    - Inputs: If product exists (false branch)
    - Outputs: Shopify, Create SetVariant
    - Failure Modes: User errors, auth failures.
  - **Shopify, Create SetVariant**
    - Type: GraphQL Mutation
    - Role: Bulk update variant details for the newly created product.
    - Configuration: Product and variant IDs from creation response; variant data from Loop.
    - Inputs: Shopify, CreateProduct
    - Outputs: Shopify, Create SetInventory
    - Failure Modes: Mutation errors, variant creation issues.
  - **Shopify, Create SetInventory**
    - Type: GraphQL Mutation
    - Role: Sets inventory quantity for newly created variant at store location.
    - Configuration: Inventory item ID and location ID from previous nodes; quantity from Loop.
    - Inputs: Shopify, Create SetVariant
    - Outputs: Airtable, MarkRecord as Sync'd
    - Failure Modes: Permissions, inventory item unavailable.
  - **Airtable, MarkRecord as Sync'd**
    - See above in update path.

#### 1.7 Airtable Sync Flag Update

- **Overview:** After each product create or update, marks the Airtable record’s `sync` field as false to prevent re-sync.
- **Nodes Involved:** Airtable, MarkRecord as Sync'd
- **Node Details:**
  - Type: Airtable Update
  - Role: Set `sync` field false on Airtable record.
  - Configuration: Uses record ID from Loop item for update.
  - Inputs: From Shopify SetInventory nodes.
  - Outputs: Loop (to continue processing)
  - Failure Modes: Airtable API errors, invalid record ID.

#### 1.8 Workflow Completion

- **Overview:** Ends the current batch of processing or workflow execution.
- **Nodes Involved:** Finished
- **Node Details:**
  - Type: No Operation (NoOp)
  - Role: Acts as a terminal node for the Loop’s false or empty output.
  - Configuration: None.
  - Inputs: Loop (secondary output)
  - Outputs: None.
  - Failure Modes: None.

---

### 3. Summary Table

| Node Name                  | Node Type            | Functional Role                                | Input Node(s)                | Output Node(s)                               | Sticky Note                                                                                                                |
|----------------------------|----------------------|-----------------------------------------------|-----------------------------|----------------------------------------------|----------------------------------------------------------------------------------------------------------------------------|
| Start Workflow             | Manual Trigger       | Manual workflow start                          | None                        | Shopify, GetLocations                        |                                                                                                                            |
| Shopify, GetLocations      | GraphQL              | Fetch store location for inventory operations | Start Workflow              | Airtable, FetchRecords                       |                                                                                                                            |
| Airtable, FetchRecords     | Airtable             | Fetch Airtable products flagged for sync      | Shopify, GetLocations       | Loop                                         |                                                                                                                            |
| Loop                      | SplitInBatches       | Process each Airtable product record individually | Airtable, FetchRecords      | Finished, Shopify, ProductQuery              |                                                                                                                            |
| Finished                  | NoOp                 | Workflow termination                           | Loop                        | None                                         |                                                                                                                            |
| Shopify, ProductQuery     | GraphQL              | Query Shopify product by handle (slug)         | Loop                        | If product exists                            |                                                                                                                            |
| If product exists         | If                   | Branch based on product existence in Shopify   | Shopify, ProductQuery       | Shopify, UpdateProduct; Shopify, CreateProduct |                                                                                                                            |
| Shopify, UpdateProduct    | GraphQL              | Update existing Shopify product metadata       | If product exists (true)    | Shopify, Update SetVariant                   |                                                                                                                            |
| Shopify, Update SetVariant| GraphQL              | Bulk update variants of existing product       | Shopify, UpdateProduct      | Shopify, Update SetInventory                 |                                                                                                                            |
| Shopify, Update SetInventory | GraphQL           | Set inventory quantity for updated variant     | Shopify, Update SetVariant  | Airtable, MarkRecord as Sync'd               |                                                                                                                            |
| Shopify, CreateProduct    | GraphQL              | Create new Shopify product                      | If product exists (false)   | Shopify, Create SetVariant                   |                                                                                                                            |
| Shopify, Create SetVariant| GraphQL              | Bulk update variants for new product            | Shopify, CreateProduct      | Shopify, Create SetInventory                 |                                                                                                                            |
| Shopify, Create SetInventory| GraphQL            | Set inventory quantity for new variant          | Shopify, Create SetVariant  | Airtable, MarkRecord as Sync'd               |                                                                                                                            |
| Airtable, MarkRecord as Sync'd | Airtable         | Mark Airtable record `sync` field as false      | Shopify, Update SetInventory; Shopify, Create SetInventory | Loop                              |                                                                                                                            |
| Sticky Note               | Sticky Note          | Explanation and setup notes                      | None                        | None                                         | ## Create Products in Shopify from Airtable... [full content in section 5]                                                  |
| Sticky Note2              | Sticky Note          | Summary: Update or create products & inventory | None                        | None                                         | ## Shopify, Product Update or Create as required...                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node**
   - Name: `Start Workflow`
   - Purpose: Manual start of the workflow.

2. **Create Shopify GraphQL node to fetch locations**
   - Name: `Shopify, GetLocations`
   - Credentials: Shopify GraphQL Header Auth (with your store’s API token)
   - Endpoint: `https://[your-shopify-store].myshopify.com/admin/api/2025-04/graphql.json`
   - Query:
     ```graphql
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
   - Connect `Start Workflow` output to this node.
   - Set `Execute Once` to true.

3. **Create Airtable node to fetch records**
   - Name: `Airtable, FetchRecords`
   - Credentials: Airtable Personal Access Token
   - Base: Your Airtable base ID
   - Table: Your products table ID
   - Operation: Search
   - Fields to fetch: title, description, company, status, slug, price, compare_at_price, sku, stock_on_hand, type
   - Filter Formula: `sync` (only records where `sync` is true)
   - Connect `Shopify, GetLocations` output to this node.

4. **Create SplitInBatches node**
   - Name: `Loop`
   - Default settings (batch size 1)
   - Connect `Airtable, FetchRecords` output to this node.

5. **Create Shopify GraphQL node to query product by handle**
   - Name: `Shopify, ProductQuery`
   - Credentials: Shopify GraphQL Header Auth
   - Endpoint: Shopify 2025-04 GraphQL Admin API URL
   - Query:
     ```graphql
     query Products ($handle: String!) {
       productByHandle(handle: $handle) {
         id
         title
         variants(first:20) {
           edges {
             node {
               id
               title
               sku
               price
               inventoryItem {
                 id
               }
             }
           }
         }
         updatedAt
         createdAt
       }
     }
     ```
   - Variables:
     ```json
     {
       "handle": "={{ $json.slug }}"
     }
     ```
   - Connect `Loop` output to this node.

6. **Create If node to check product existence**
   - Name: `If product exists`
   - Condition: Check if `{{$json.data.productByHandle}}` exists
   - Connect `Shopify, ProductQuery` output to this node.

7. **Create Shopify GraphQL node to update product**
   - Name: `Shopify, UpdateProduct`
   - Credentials: Shopify GraphQL Header Auth
   - Endpoint: Shopify API URL
   - Mutation:
     ```graphql
     mutation productUpdate($product: ProductUpdateInput, $media: [CreateMediaInput!]) {
       productUpdate(product: $product, media: $media) {
         product {
           id
           title
           descriptionHtml
           vendor
           productType
           status
           handle
           tracksInventory
           variants(first:10) {
             edges {
               node {
                 id
                 sku
                 displayName
                 inventoryItem {
                   id
                   sku
                   tracked
                   requiresShipping
                 }
               }
             }
           }
         }
         userErrors {
           field
           message
         }
       }
     }
     ```
   - Variables (use expressions to pull data from Loop and Shopify product query):
     ```json
     {
       "product": {
         "id": "={{$json.data.productByHandle.id}}",
         "title": "={{$('Loop').item.json.title}}",
         "descriptionHtml": "={{$('Loop').item.json.description}}",
         "vendor": "={{$('Loop').item.json.company}}",
         "productType": "={{$('Loop').item.json.type}}",
         "status": "={{$('Loop').item.json.status}}",
         "handle": "={{$('Loop').item.json.slug}}"
       },
       "media": [{
         "alt": "alt tag",
         "mediaContentType": "IMAGE",
         "originalSource": "https://placehold.co/800x600.png"
       }]
     }
     ```
   - Connect `If product exists` true output to this node.

8. **Create Shopify GraphQL node to bulk update variants (update path)**
   - Name: `Shopify, Update SetVariant`
   - Credentials: Shopify GraphQL Header Auth
   - Mutation:
     ```graphql
     mutation productVariantsBulkUpdate($productId: ID!, $variants: [ProductVariantsBulkInput!]!) {
       productVariantsBulkUpdate(productId: $productId, variants: $variants) {
         product {
           id
           handle
         }
         productVariants {
           id
           sku
           title
           price
           compareAtPrice
         }
         userErrors {
           field
           message
         }
       }
     }
     ```
   - Variables:
     ```json
     {
       "productId": "={{ $json.data.productUpdate.product.id }}",
       "variants": {
         "id": "={{ $json.data.productUpdate.product.variants.edges[0].node.id }}",
         "price": "={{ $('Loop').item.json.price }}",
         "compareAtPrice": "={{ $('Loop').item.json.compare_at_price }}",
         "inventoryItem": {
           "sku": "={{ $('Loop').item.json.sku }}",
           "tracked": true,
           "requiresShipping": true
         }
       }
     }
     ```
   - Connect `Shopify, UpdateProduct` output to this node.

9. **Create Shopify GraphQL node to set inventory (update path)**
   - Name: `Shopify, Update SetInventory`
   - Credentials: Shopify GraphQL Header Auth
   - Mutation:
     ```graphql
     mutation inventorySetOnHandQuantities($input: InventorySetOnHandQuantitiesInput!) {
       inventorySetOnHandQuantities(input: $input) {
         userErrors {
           field
           message
         }
       }
     }
     ```
   - Variables:
     ```json
     {
       "input": {
         "reason": "correction",
         "setQuantities": [{
           "inventoryItemId": "={{ $json.data.productUpdate.product.variants.edges[0].node.inventoryItem.id }}",
           "locationId": "={{ $('Shopify, GetLocations').item.json.data.locations.edges[0].node.id }}",
           "quantity": "={{ $('Loop').item.json.stock_on_hand }}"
         }]
       }
     }
     ```
   - Connect `Shopify, Update SetVariant` output to this node.

10. **Create Shopify GraphQL node to create product (creation path)**
    - Name: `Shopify, CreateProduct`
    - Credentials: Shopify GraphQL Header Auth
    - Mutation:
      ```graphql
      mutation productCreate($product: ProductCreateInput!, $media: [CreateMediaInput!]) {
        productCreate(product: $product, media: $media) {
          product {
            id
            title
            descriptionHtml
            vendor
            productType
            status
            handle
            variants(first:10) {
              edges {
                node {
                  id
                  sku
                  displayName
                  inventoryItem {
                    id
                    sku
                    tracked
                    requiresShipping
                  }
                }
              }
            }
            media(first: 10) {
              edges {
                node {
                  alt
                  mediaContentType
                  status
                  id
                  preview {
                    image {
                      url
                    }
                    status
                  }
                }
              }
            }
            options {
              id
              name
              position
              optionValues {
                id
                name
                hasVariants
              }
            }
          }
          userErrors {
            field
            message
          }
        }
      }
      ```
    - Variables:
      ```json
      {
        "product": {
          "title": "={{$('Loop').item.json.title}}",
          "descriptionHtml": "={{$('Loop').item.json.description}}",
          "vendor": "={{$('Loop').item.json.company}}",
          "productType": "={{$('Loop').item.json.type}}",
          "status": "={{$('Loop').item.json.status}}",
          "handle": "={{$('Loop').item.json.slug}}"
        },
        "media": [{
          "alt": "alt tag",
          "mediaContentType": "IMAGE",
          "originalSource": "https://placehold.co/800x600.png"
        }]
      }
      ```
    - Connect `If product exists` false output to this node.

11. **Create Shopify GraphQL node to bulk update variants (creation path)**
    - Name: `Shopify, Create SetVariant`
    - Credentials: Shopify GraphQL Header Auth
    - Same mutation as `Shopify, Update SetVariant` but variables sourced from product creation response:
      ```json
      {
        "productId": "={{ $json.data.productCreate.product.id }}",
        "variants": {
          "id": "={{ $json.data.productCreate.product.variants.edges[0].node.id }}",
          "price": "={{ $('Loop').item.json.price }}",
          "compareAtPrice": "={{ $('Loop').item.json.compare_at_price }}",
          "inventoryItem": {
            "sku": "={{ $('Loop').item.json.sku }}",
            "tracked": true
          }
        }
      }
      ```
    - Connect `Shopify, CreateProduct` output to this node.

12. **Create Shopify GraphQL node to set inventory (creation path)**
    - Name: `Shopify, Create SetInventory`
    - Credentials: Shopify GraphQL Header Auth
    - Same mutation as Update SetInventory, variables sourced from creation nodes:
      ```json
      {
        "input": {
          "reason": "correction",
          "setQuantities": [{
            "inventoryItemId": "={{ $json.data.productCreate.product.variants.edges[0].node.inventoryItem.id }}",
            "locationId": "={{ $('Shopify, GetLocations').item.json.data.locations.edges[0].node.id }}",
            "quantity": "={{ $('Loop').item.json.stock_on_hand }}"
          }]
        }
      }
      ```
    - Connect `Shopify, Create SetVariant` output to this node.

13. **Create Airtable Update node to mark record as synced**
    - Name: `Airtable, MarkRecord as Sync'd`
    - Credentials: Airtable Personal Access Token
    - Base and Table: Same as FetchRecords
    - Operation: Update
    - Mapping:
      - Match by record ID from `Loop` item JSON `id`
      - Set `sync` field to `false`
    - Connect outputs from both `Shopify, Update SetInventory` and `Shopify, Create SetInventory` nodes to this node.
    - Connect this node back to `Loop` to continue the batch processing.

14. **Create NoOp node for workflow finish**
    - Name: `Finished`
    - Connect the secondary output of `Loop` to this node to handle empty or finished batches.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| ## Create Products in Shopify from Airtable<br><br>This workflow creates products in your Shopify store from Airtable. It also enables inventory tracking and sets the quantity of an inventory item at your store's default location. <br><br>This is a great way to keep Shopify in sync with Airtable if Airtable is your primary source of data. Only records with the 'sync' column set to true are synced. Setup Airtable Automation so that if any records are created or updated then this flag is set to true.<br><br>Records are matched using the 'slug' column which Shopify calls a handle.<br><br>### Airtable Setup Notes<br><br>The Airtable products table has the following columns:<br>- title - free text<br>- description - free text<br>- company - free text<br>- type - free text<br>- status - ACTIVE, DRAFT or ARCHIVE<br>- slug - used in the product url, text with no spaces, can also use hyphen.<br>- price - sale price of the products<br>- compare_at_price - compare at price for products<br>- sku - unique code for each product<br>- stock_on_hand - quantity of this item available for purchase.<br>- sync - boolean, set to true to sync this record.<br><br>### Update GraphQL nodes with your Shopify store URL<br><br>1) Replace the URL in all GraphQL nodes with the URL for your Shopify store.<br>2) All GraphQL requests use the Shopify 2025-04 GraphQL Admin API. | Workflow description note embedded in Sticky Note node (positioned top-left)                      |
| ## Shopify, Product Update or Create as required<br>- Create or Update product.<br>- Enable inventory tracking<br>- Set inventory quantity                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Sticky Note near Shopify product mutation nodes summarizing the primary workflow purpose          |

---

**Disclaimer:** The provided text is extracted exclusively from an n8n workflow automation. It complies fully with content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.
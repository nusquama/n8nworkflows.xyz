Automatically Correct Wrong Shipping Addresses in Billbee Orders

https://n8nworkflows.xyz/workflows/automatically-correct-wrong-shipping-addresses-in-billbee-orders-2609


# Automatically Correct Wrong Shipping Addresses in Billbee Orders

### 1. Workflow Overview

This workflow automates the validation and correction of shipping addresses for orders in Billbee, an order management platform for e-commerce businesses. Its main purpose is to ensure accurate delivery information by integrating Billbee with the Endereco address validation API. It is designed to reduce manual effort, minimize shipping errors, and improve order fulfillment efficiency.

The workflow logic is organized into the following blocks:

- **1.1 Trigger and Initialization**  
  Receives order import events via Billbee webhook and initializes necessary API keys and parameters.

- **1.2 Fetching and Preparing Order Data**  
  Retrieves detailed order shipping address data from Billbee and maps required fields for validation.

- **1.3 Address Filtering and House Number Extraction**  
  Applies filters to exclude pickup locations and processes house number information, including special cases where house number data might be embedded in a secondary address line.

- **1.4 Address Validation via Endereco API**  
  Sends the prepared address data to the Endereco API for validation and receives possible corrected address suggestions.

- **1.5 Conditional Processing Based on Validation Result**  
  Depending on whether the Endereco API returns a corrected address or not, updates the shipping address in Billbee or tags the order for manual review.

- **1.6 Tagging and Finalization**  
  Adds appropriate tags to the Billbee order indicating success or failure of address validation for tracking purposes.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger and Initialization

- **Overview:**  
  This block starts the workflow by receiving an order import webhook from Billbee and sets up API credentials and initial parameters.

- **Nodes Involved:**  
  - Webhook  
  - ConfigNode  
  - Wait

- **Node Details:**  

  - **Webhook**  
    - Type: Webhook  
    - Role: Entry point triggered by Billbee when an order is imported.  
    - Config: Receives HTTP requests with query parameter `Id` representing the Billbee Order ID.  
    - Input: External HTTP call from Billbee.  
    - Output: Passes query parameters downstream.  
    - Edge cases: Missing or malformed `Id` parameter could cause failure in subsequent nodes.

  - **ConfigNode**  
    - Type: Set  
    - Role: Stores API keys and extracts the order ID from webhook data.  
    - Configuration:  
      - `X-Billbee-Api-Key`: Billbee Developer API Key (to be configured by user).  
      - `X-Auth-Key-Endereco`: Endereco API Key (to be configured by user).  
      - `orderID`: Set from webhook query parameter `Id`.  
    - Input: Webhook output.  
    - Output: Passes API keys and order ID downstream.  
    - Edge cases: Missing or invalid keys will cause authentication errors.

  - **Wait**  
    - Type: Wait  
    - Role: Introduces a 1 second delay before proceeding to avoid rate-limiting or timing issues.  
    - Input: ConfigNode output.  
    - Output: Proceeds to data fetching node.

---

#### 1.2 Fetching and Preparing Order Data

- **Overview:**  
  Retrieves order details from Billbee using the order ID and extracts relevant shipping address fields for validation.

- **Nodes Involved:**  
  - get order data (HTTP Request)  
  - Split Out Order Data (SplitOut)  
  - Set Address Fields (Set)  
  - Filter Out PickUpShops (Filter)  
  - check if housenumer is not empty (If)  

- **Node Details:**  

  - **get order data**  
    - Type: HTTP Request  
    - Role: Calls Billbee API to fetch order details using the order ID.  
    - Config:  
      - URL: `https://api.billbee.io/api/v1/orders/{{ $json.orderID }}`  
      - Authentication: Basic HTTP Auth with Billbee API Key in header `X-Billbee-Api-Key`.  
    - Input: Output from Wait node.  
    - Output: JSON containing full order data.  
    - Edge cases: HTTP errors, authentication failure, invalid order ID.

  - **Split Out Order Data**  
    - Type: SplitOut  
    - Role: Extracts shipping address fields from nested JSON structure for easier access.  
    - Config: Splits fields such as `BillbeeId`, `FirstName`, `LastName`, `Street`, `HouseNumber`, `Zip`, `City`, `CountryISO2`, `Line2`, and `NameAddition`.  
    - Input: Output of get order data node.  
    - Output: Individual JSON fields for address components.  
    - Edge cases: Missing or malformed fields.

  - **Set Address Fields**  
    - Type: Set  
    - Role: Maps and normalizes extracted address data into a flat structure for validation; cleans house number by removing slashes.  
    - Key expressions:  
      - `housenumber` is set as `HouseNumber` with any `/` removed.  
      - Billbee Order ID and Shipping Address ID are captured for later updates.  
    - Input: Split Out Order Data output.  
    - Output: Structured address fields ready for filtering.  

  - **Filter Out PickUpShops**  
    - Type: Filter  
    - Role: Excludes orders where the street contains pickup shop keywords (`Postfiliale`, `Packstation`, `Paketshop`) to avoid validating non-delivery addresses.  
    - Input: Set Address Fields output.  
    - Output: Orders passing filter proceed; others are excluded from validation.  
    - Edge cases: Case sensitivity or incomplete keyword matching might affect filtering accuracy.

  - **check if housenumer is not empty**  
    - Type: If  
    - Role: Checks if the house number field is present and non-empty, a critical validation step before calling the API.  
    - Input: Filter Out PickUpShops output.  
    - Output:  
      - True: Proceeds to address validation.  
      - False: Proceeds to check if `AddressLine2` contains a number for alternative house number extraction.

---

#### 1.3 Address Filtering and House Number Extraction

- **Overview:**  
  Handles cases where the house number might be embedded in the secondary address line (`AddressLine2`) either as a pure number or alphanumeric.

- **Nodes Involved:**  
  - check if addressline 2 contains number (If)  
  - set value of addressline2 as housenumber (Set)  
  - check if addressline 2 contains number and letter (If)  
  - set value of addressline2 as housenumber number+letter (Set)  
  - set billbee tag manual check (HTTP Request)

- **Node Details:**  

  - **check if addressline 2 contains number**  
    - Type: If  
    - Role: Checks if `AddressLine2` is numeric (contains only numbers).  
    - Input: False branch of house number check or earlier nodes.  
    - Output:  
      - True: Assigns `AddressLine2` as house number (numeric).  
      - False: Checks for alphanumeric content next.

  - **set value of addressline2 as housenumber**  
    - Type: Set  
    - Role: Sets `housenumber` field to the numeric `AddressLine2` value.  
    - Input: True branch of numeric check.  
    - Output: Proceeds to address validation.

  - **check if addressline 2 contains number and letter**  
    - Type: If  
    - Role: Checks if `AddressLine2` contains both letters and numbers (alphanumeric).  
    - Input: False branch of numeric check.  
    - Output:  
      - True: Uses alphanumeric `AddressLine2` as house number.  
      - False: Tags order for manual address check.

  - **set value of addressline2 as housenumber number+letter**  
    - Type: Set  
    - Role: Assigns alphanumeric `AddressLine2` to `housenumber` field.  
    - Input: True branch from alphanumeric check.  
    - Output: Proceeds to address validation.

  - **set billbee tag manual check**  
    - Type: HTTP Request  
    - Role: Adds a "manual_address_check" tag to the Billbee order to flag it for manual review when house number cannot be inferred.  
    - API URL: `https://api.billbee.io/api/v1/orders/{{ BillbeeID }}/tags`  
    - Authentication: Billbee API Key in header.  
    - Input: False branch of alphanumeric check.  
    - Output: Ends workflow for this order.  
    - Edge cases: API failures or network issues.

---

#### 1.4 Address Validation via Endereco API

- **Overview:**  
  Sends the cleaned and prepared address to the Endereco API for validation and potential correction.

- **Nodes Involved:**  
  - Check Address endereco api (HTTP Request)  
  - check if new address is not empty (If)  
  - Split Out Corrected Address (SplitOut)  
  - Wait1

- **Node Details:**  

  - **Check Address endereco api**  
    - Type: HTTP Request  
    - Role: Calls Endereco JSON-RPC API method `addressCheck` with country, language, postal code, city, street, and house number parameters.  
    - Config:  
      - Method: POST  
      - Headers: `X-Auth-Key` (Endereco API Key), `Content-Type: application/json`, transaction-related headers.  
      - Body: JSON-RPC with address parameters from previous Set node.  
    - Input: Prepared address data from prior steps.  
    - Output: API response with validation results and suggested corrections.  
    - Edge cases: API key invalid, network timeouts, or invalid input data.

  - **check if new address is not empty**  
    - Type: If  
    - Role: Checks if the response contains any predictions (corrected addresses).  
    - Input: Endereco API response.  
    - Output:  
      - True: Proceed to extract corrected address.  
      - False: Tag order as validation failed.

  - **Split Out Corrected Address**  
    - Type: SplitOut  
    - Role: Extracts the first prediction from the `result.predictions` array for use in updating Billbee.  
    - Input: True branch of previous if node.  
    - Output: Passes corrected address data downstream.

  - **Wait1**  
    - Type: Wait  
    - Role: Introduces a 1 second delay before updating Billbee to comply with rate limits or ensure API readiness.  
    - Input: Corrected address output.  
    - Output: Proceeds to update Billbee address.

---

#### 1.5 Conditional Processing Based on Validation Result

- **Overview:**  
  Updates Billbee with the corrected address if available or tags the order with an error tag if validation failed.

- **Nodes Involved:**  
  - set new delivery address to billbee (HTTP Request)  
  - set billbee tag (HTTP Request)  
  - set billbee success (HTTP Request)

- **Node Details:**  

  - **set new delivery address to billbee**  
    - Type: HTTP Request  
    - Role: Updates the shipping address in Billbee with corrected address fields from Endereco.  
    - Configuration:  
      - Method: PATCH  
      - URL: `https://api.billbee.io/api/v1/customers/addresses/{{ BillbeeShippingAddressID }}`  
      - Body: Contains corrected `Housenumber`, `Street`, `Zip`, and `City`.  
      - Authentication: Billbee API Key in header.  
    - Input: Corrected address after wait.  
    - Output: Proceeds to tag order as validated.

  - **set billbee tag**  
    - Type: HTTP Request  
    - Role: Adds tag `endereco_address_failed` to Billbee orders where no correction was suggested by Endereco API.  
    - Method: POST  
    - URL: `https://api.billbee.io/api/v1/orders/{{ BillbeeID }}/tags`  
    - Body: JSON with tag array containing `"endereco_address_failed"`.  
    - Authentication: Billbee API Key.  
    - Input: False branch when no corrected address is found.  
    - Output: Ends workflow for failed validation.

  - **set billbee success**  
    - Type: HTTP Request  
    - Role: Adds tag `endereco_address_validated` to Billbee orders after successful address update.  
    - Method: POST  
    - URL: `https://api.billbee.io/api/v1/orders/{{ BillbeeID }}/tags`  
    - Body: JSON with tag array containing `"endereco_address_validated"`.  
    - Authentication: Billbee API Key.  
    - Input: Success branch after address update.  
    - Output: Ends workflow for successfully processed order.

---

#### 1.6 Tagging and Finalization

- This is integrated as part of the conditional processing above with tagging nodes to track validation status.

---

### 3. Summary Table

| Node Name                      | Node Type          | Functional Role                                   | Input Node(s)                   | Output Node(s)                  | Sticky Note                                                                                                                                |
|--------------------------------|--------------------|-------------------------------------------------|--------------------------------|--------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| Webhook                        | Webhook            | Entry trigger receiving Billbee order import    | -                              | ConfigNode                     |                                                                                                                                            |
| ConfigNode                    | Set                | Sets API keys and extracts order ID             | Webhook                        | Wait                          |                                                                                                                                            |
| Wait                          | Wait               | Delay to avoid rate limits                        | ConfigNode                    | get order data                |                                                                                                                                            |
| get order data                | HTTP Request       | Fetches order data from Billbee API              | Wait                          | Split Out Order Data           |                                                                                                                                            |
| Split Out Order Data          | SplitOut           | Extracts shipping address fields                  | get order data                | Set Address Fields             |                                                                                                                                            |
| Set Address Fields            | Set                | Normalizes and maps address data                  | Split Out Order Data          | Filter Out PickUpShops         | ## Get and Prepare Oder Data                                                                                                               |
| Filter Out PickUpShops        | Filter             | Excludes pickup shops from validation             | Set Address Fields            | check if housenumer is not empty | ## Include Filter You want to filter out PickUp Shops or Parcel Stations for example in Germany: "Postfiliale, Paketshop, Packstation" This... |
| check if housenumer is not empty | If                 | Checks if house number exists                      | Filter Out PickUpShops        | Check Address endereco api, check if addressline 2 contains number | ## House Number Validation                                                                                                                |
| check if addressline 2 contains number | If                 | Checks if AddressLine2 is numeric                  | check if housenumer is not empty | set value of addressline2 as housenumber, check if addressline 2 contains number and letter |                                                                                                                                            |
| set value of addressline2 as housenumber | Set                | Sets house number from numeric AddressLine2       | check if addressline 2 contains number | Check Address endereco api   |                                                                                                                                            |
| check if addressline 2 contains number and letter | If                 | Checks if AddressLine2 contains letters and numbers | check if addressline 2 contains number | set value of addressline2 as housenumber number+letter, set billbee tag manual check |                                                                                                                                            |
| set value of addressline2 as housenumber number+letter | Set                | Sets house number from alphanumeric AddressLine2  | check if addressline 2 contains number and letter | Check Address endereco api   |                                                                                                                                            |
| set billbee tag manual check  | HTTP Request       | Tags order for manual address check                | check if addressline 2 contains number and letter (false branch) | -                             |                                                                                                                                            |
| Check Address endereco api    | HTTP Request       | Sends address to Endereco API for validation       | check if housenumer is not empty, set value of addressline2 as housenumber, set value of addressline2 as housenumber number+letter | check if new address is not empty | ## Address Validation & Correction                                                                                                         |
| check if new address is not empty | If                 | Checks if Endereco API returned corrected address | Check Address endereco api    | Split Out Corrected Address, set billbee tag |                                                                                                                                            |
| Split Out Corrected Address   | SplitOut           | Extracts corrected address from Endereco response | check if new address is not empty | Wait1                         |                                                                                                                                            |
| Wait1                         | Wait               | Delay before updating Billbee                       | Split Out Corrected Address   | set new delivery address to billbee |                                                                                                                                            |
| set new delivery address to billbee | HTTP Request       | Updates shipping address in Billbee                 | Wait1                         | set billbee success            |                                                                                                                                            |
| set billbee tag               | HTTP Request       | Tags order with "endereco_address_failed"          | check if new address is not empty (false branch) | -                             |                                                                                                                                            |
| set billbee success           | HTTP Request       | Tags order with "endereco_address_validated"       | set new delivery address to billbee | -                             |                                                                                                                                            |
| Sticky Note                   | Sticky Note        | Comment: "## Get and Prepare Oder Data"            | -                            | -                             | ## Get and Prepare Oder Data                                                                                                               |
| Sticky Note1                  | Sticky Note        | Workflow overview, requirements, benefits          | -                            | -                             | # **Address Validation Workflow** ... (full workflow description and benefits)                                                            |
| Sticky Note2                  | Sticky Note        | API documentation links                             | -                            | -                             | ## API Docs Endereco: https://github.com/Endereco/enderecoservice_api Billbee: https://app.billbee.io//swagger/ui/index                   |
| Sticky Note3                  | Sticky Note        | Billbee automation rule setup instructions          | -                            | -                             | ### Bilbee Setup ... (rule configuration and trigger details)                                                                             |
| Sticky Note4                  | Sticky Note        | Filter explanation for excluding pickup shops       | -                            | -                             | ## Include Filter ... (explanation on filtering pickup shops)                                                                             |
| Sticky Note5                  | Sticky Note        | Attention note on house number validation block      | -                            | -                             | ## Open ME!                                                                                                                                 |
| Sticky Note6                  | Sticky Note        | House number validation context                      | -                            | -                             | ## House Number Validation                                                                                                                |
| Sticky Note7                  | Sticky Note        | Address validation and correction context            | -                            | -                             | ## Address Validation & Correction                                                                                                         |
| Sticky Note8                  | Sticky Note        | Setup instructions for credentials                   | -                            | -                             | ### **Setup** Create Basic Auth for BillbeeUser, paste API keys                                                                           |
| Sticky Note9                  | Sticky Note        | Setup instructions for Basic Auth selection          | -                            | -                             | ### **Setup** Select your BillbeeUser Basic Auth                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook node:**  
   - Set path to a unique identifier (e.g., `786e8a93-9837-44e6-81ae-a173ce25a14f`).  
   - This node will receive HTTP GET requests from Billbee with query parameter `Id` (order ID).  
   - No authentication required here.

2. **Add a Set node named "ConfigNode":**  
   - Create three fields:  
     - `X-Billbee-Api-Key` (string): Paste your Billbee Developer API Key.  
     - `X-Auth-Key-Endereco` (string): Paste your Endereco API Key.  
     - `orderID` (string): Expression `{{$json.query.Id}}` to extract order ID from webhook query.  
   - Connect Webhook → ConfigNode.

3. **Add a Wait node ("Wait"):**  
   - Set delay to 1 second (to avoid rate limiting).  
   - Connect ConfigNode → Wait.

4. **Add HTTP Request node "get order data":**  
   - Method: GET  
   - URL: `https://api.billbee.io/api/v1/orders/{{ $json.orderID }}`  
   - Authentication: HTTP Basic Auth configured with Billbee User credentials (email as username, User API key as password).  
   - Header: Add `X-Billbee-Api-Key` with value from `{{$json["X-Billbee-Api-Key"]}}`.  
   - Connect Wait → get order data.

5. **Add SplitOut node "Split Out Order Data":**  
   - Field to split out:  
     - `Data.ShippingAddress.BillbeeId`  
     - `Data.ShippingAddress.FirstName`  
     - `Data.ShippingAddress.LastName`  
     - `Data.ShippingAddress.Street`  
     - `Data.ShippingAddress.HouseNumber`  
     - `Data.ShippingAddress.Zip`  
     - `Data.ShippingAddress.City`  
     - `Data.ShippingAddress.CountryISO2`  
     - `Data.ShippingAddress.Line2`  
     - `Data.ShippingAddress.NameAddition`  
   - Connect get order data → Split Out Order Data.

6. **Add Set node "Set Address Fields":**  
   - Map the fields extracted above into new keys:  
     - `first_name`: `{{$json["Data.ShippingAddress.FirstName"]}}`  
     - `family_name`: `{{$json["Data.ShippingAddress.LastName"]}}`  
     - `Street`: `{{$json["Data.ShippingAddress.Street"]}}`  
     - `housenumber`: `{{$json["Data.ShippingAddress.HouseNumber"].replace("/", "")}}` (remove any slashes)  
     - `zip`: `{{$json["Data.ShippingAddress.Zip"]}}`  
     - `location`: `{{$json["Data.ShippingAddress.City"]}}`  
     - `BillbeeID`: `{{$json.query.Id}}` (from webhook)  
     - `BillbeeShippingAddressID`: `{{$json["Data.ShippingAddress.BillbeeId"]}}`  
     - `CountryISO2`: `{{$json["Data.ShippingAddress.CountryISO2"]}}`  
     - `AddressLine2`: `{{$json["Data.ShippingAddress.Line2"]}}`  
     - `NameAddition`: `{{$json["Data.ShippingAddress.NameAddition"]}}`  
   - Connect Split Out Order Data → Set Address Fields.

7. **Add Filter node "Filter Out PickUpShops":**  
   - Condition: Exclude if `Street` contains any of: "Postfiliale", "Packstation", "Paketshop" (use multiple NOT CONTAINS OR conditions).  
   - Connect Set Address Fields → Filter Out PickUpShops.

8. **Add If node "check if housenumer is not empty":**  
   - Condition: Check if `housenumber` is not empty.  
   - Connect Filter Out PickUpShops → check if housenumer is not empty.

9. **True branch of house number check:**  
   - Connect to HTTP Request node "Check Address endereco api":  
     - Method: POST  
     - URL: `https://endereco-service.de/rpc/v1`  
     - Headers:  
       - `X-Auth-Key`: `{{$json["X-Auth-Key-Endereco"]}}`  
       - `Content-Type`: `application/json`  
       - Plus transaction headers as static values.  
     - Body (JSON):  
       ```json
       {
         "jsonrpc": "2.0",
         "id": 1,
         "method": "addressCheck",
         "params": {
           "country": "{{ $json.CountryISO2 }}",
           "language": "{{ $json.CountryISO2 }}",
           "postCode": "{{ $json.zip }}",
           "cityName": "{{ $json.location }}",
           "street": "{{ $json.Street }}",
           "houseNumber": "{{ $json.housenumber }}"
         }
       }
       ```  
     - Connect check if housenumer is not empty (true) → Check Address endereco api.

10. **False branch of house number check:**  
    - Add If node "check if addressline 2 contains number":  
      - Condition: Check if `AddressLine2` is numeric (use `isNumeric()` JS expression).  
    - Connect check if housenumer is not empty (false) → check if addressline 2 contains number.

11. **True branch of numeric check:**  
    - Add Set node "set value of addressline2 as housenumber":  
      - Set `housenumber` to `AddressLine2`.  
    - Connect check if addressline 2 contains number (true) → set value of addressline2 as housenumber.

12. **Connect set value of addressline2 as housenumber → Check Address endereco api.**

13. **False branch of numeric check:**  
    - Add If node "check if addressline 2 contains number and letter":  
      - Condition: Use regex to check if `AddressLine2` contains both letters and numbers.  
    - Connect check if addressline 2 contains number (false) → check if addressline 2 contains number and letter.

14. **True branch of alphanumeric check:**  
    - Add Set node "set value of addressline2 as housenumber number+letter":  
      - Set `housenumber` to `AddressLine2`.  
    - Connect check if addressline 2 contains number and letter (true) → set value of addressline2 as housenumber number+letter.

15. **Connect set value of addressline2 as housenumber number+letter → Check Address endereco api.**

16. **False branch of alphanumeric check:**  
    - Add HTTP Request node "set billbee tag manual check":  
      - Method: POST  
      - URL: `https://api.billbee.io/api/v1/orders/{{ BillbeeID }}/tags`  
      - Body: `{ "Tags": ["manual_address_check"] }`  
      - Authentication: Billbee API Key.  
    - Connect check if addressline 2 contains number and letter (false) → set billbee tag manual check.

17. **Add If node "check if new address is not empty":**  
    - Condition: Check if `result.predictions` array in Endereco API response is not empty.  
    - Connect Check Address endereco api → check if new address is not empty.

18. **True branch of new address check:**  
    - Add SplitOut node "Split Out Corrected Address":  
      - Field to split out: `result.predictions` (extract corrected address).  
    - Connect check if new address is not empty (true) → Split Out Corrected Address.

19. **Add Wait node "Wait1":**  
    - Set delay to 1 second before updating Billbee.  
    - Connect Split Out Corrected Address → Wait1.

20. **Add HTTP Request node "set new delivery address to billbee":**  
    - Method: PATCH  
    - URL: `https://api.billbee.io/api/v1/customers/addresses/{{ BillbeeShippingAddressID }}`  
    - Body Parameters:  
      - `Housenumber`: `{{$json.houseNumber}}`  
      - `Street`: `{{$json.street}}`  
      - `Zip`: `{{$json.postCode}}`  
      - `City`: `{{$json.cityName}}`  
    - Authentication: Billbee API Key.  
    - Connect Wait1 → set new delivery address to billbee.

21. **Add HTTP Request node "set billbee success":**  
    - Method: POST  
    - URL: `https://api.billbee.io/api/v1/orders/{{ BillbeeID }}/tags`  
    - Body: `{ "Tags": ["endereco_address_validated"] }`  
    - Authentication: Billbee API Key.  
    - Connect set new delivery address to billbee → set billbee success.

22. **False branch of new address check:**  
    - Add HTTP Request node "set billbee tag":  
      - Method: POST  
      - URL: `https://api.billbee.io/api/v1/orders/{{ BillbeeID }}/tags`  
      - Body: `{ "Tags": ["endereco_address_failed"] }`  
      - Authentication: Billbee API Key.  
    - Connect check if new address is not empty (false) → set billbee tag.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                 | Context or Link                                                                                                        |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------|
| Workflow automates shipping address validation for Billbee orders using Endereco API, reducing manual errors and improving fulfillment accuracy.                                             | Workflow description provided.                                                                                         |
| Billbee API key types: Developer API Key (for header) and User API Key (for Basic Auth). Basic Auth uses email as username and User API key as password.                                     | Billbee API authentication requirements.                                                                              |
| Billbee automation rule setup: trigger on "Order imported" event and call n8n webhook with `OrderId` as parameter for workflow initiation.                                                  | Sticky Note3 content.                                                                                                  |
| Endereco API documentation is available at https://github.com/Endereco/enderecoservice_api. Billbee API docs at https://app.billbee.io//swagger/ui/index                                  | Sticky Note2 content.                                                                                                  |
| Filtering excludes pickup shops such as "Postfiliale", "Paketshop", and "Packstation" to avoid validating non-delivery addresses. This can also be configured within Billbee rules.         | Sticky Note4 content.                                                                                                  |
| House number validation is a critical part of the workflow, including extracting house number from secondary address line when missing or malformed.                                        | Sticky Note6 content.                                                                                                  |
| Workflow uses small wait/delay nodes to mitigate API rate limits and avoid race conditions.                                                                                                   | Multiple wait nodes in workflow.                                                                                       |
| Suggested tags for orders: `endereco_address_validated`, `endereco_address_failed`, and `manual_address_check` to track address validation status in Billbee.                                | Tagging strategy for order tracking.                                                                                   |

---

This document provides a detailed, stepwise understanding of the "Automatically Correct Wrong Shipping Addresses in Billbee Orders" workflow, enabling both manual reproduction and informed modification of the workflow.
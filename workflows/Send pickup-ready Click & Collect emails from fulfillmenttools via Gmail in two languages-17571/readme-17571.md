Send pickup-ready Click & Collect emails from fulfillmenttools via Gmail in two languages

https://n8nworkflows.xyz/workflows/send-pickup-ready-click---collect-emails-from-fulfillmenttools-via-gmail-in-two-languages-17571


# Send pickup-ready Click & Collect emails from fulfillmenttools via Gmail in two languages

### 1. Workflow Overview

This workflow automates the notification process for Click & Collect orders that are ready for customer pickup. It listens for newly created handover jobs from fulfillmenttools, filters relevant orders based on specific sales sources and fulfillment channels, gathers comprehensive order and pickup facility data, determines the customer's language preference, and sends a localized, fully formatted HTML email via Gmail in either English or German.

The execution logic is divided into the following sequential blocks:

- **1.1 Event Reception and Validation:** Captures the handover job creation event and verifies that the source and fulfillment channel match target criteria (`FFT_Shop` and `COLLECT`).
- **1.2 Order Data Retrieval and Validation:** Queries fulfillmenttools to resolve the internal order ID from the tenant order ID, fetches the complete order profile, and checks that a valid customer email address exists.
- **1.3 Facility Lookup and Language Detection:** Fetches store and address details for the assigned pickup facility and evaluates the customer's preferred shop language (`en_US` or `de_DE`).
- **1.4 Multilingual Email Dispatch:** Routes execution to the appropriate HTML email template and dispatches the message via Gmail.

---

### 2. Block-by-Block Analysis

#### 2.1 Event Reception and Validation
- **Overview:** This block initializes the workflow upon receiving a webhook event from fulfillmenttools and filters out irrelevant traffic based on sales channel parameters.
- **Nodes Involved:**
  - `Handoverjob Created`
  - `Check If Data is Valid`
  - `No Operation, do nothing`

- **Node Details:**
  - **`Handoverjob Created`**
    - *Type and Technical Role:* `@fulfillmenttools/n8n-nodes-fulfillmenttools.fulfillmenttoolsTrigger` (Webhook Trigger).
    - *Configuration Choices:* Subscribes to the `HANDOVERJOB_CREATED` event using the subscription name `n8n Shop Pickup ready email`.
    - *Key Expressions/Variables:* Outputs incoming webhook payload data.
    - *Input/Output Connections:* Inputs: None (Entrypoint) | Outputs: `Check If Data is Valid`.
    - *Version Requirements:* v1.
    - *Edge Cases/Failure Types:* Webhook delivery failures if fulfillmenttools cannot reach the n8n instance URL; invalid or expired API credentials.
  
  - **`Check If Data is Valid`**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Conditional Router).
    - *Configuration Choices:* Evaluates two conditions combined with an `and` operator: `salesSource` equals `FFT_Shop` and `channel` equals `COLLECT`.
    - *Key Expressions/Variables:* 
      - Left Value 1: `={{ $json.payload.customAttributes.salesSource }}` (Expected: `FFT_Shop`)
      - Left Value 2: `={{ $json.payload.channel }}` (Expected: `COLLECT`)
    - *Input/Output Connections:* Inputs: `Handoverjob Created` | Outputs (True): `Get Order by tenantId` | Outputs (False): `No Operation, do nothing`.
    - *Version Requirements:* v2.3.
    - *Edge Cases/Failure Types:* Missing payload fields will cause expression evaluation errors if nested properties do not exist.

  - **`No Operation, do nothing`**
    - *Type and Technical Role:* `n8n-nodes-base.noOp` (Terminator / Fallback).
    - *Configuration Choices:* Default empty configuration.
    - *Input/Output Connections:* Inputs: `Check If Data is Valid` (False branch), `Check for valid email address` (False branch) | Outputs: None.
    - *Version Requirements:* v1.
    - *Edge Cases/Failure Types:* None.

---

#### 2.2 Order Data Retrieval and Validation
- **Overview:** This block resolves internal identifiers, fetches full order details from fulfillmenttools, and validates that a deliverable customer email address is present.
- **Nodes Involved:**
  - `Get Order by tenantId`
  - `Get Order by Id`
  - `Check for valid email address`

- **Node Details:**
  - **`Get Order by tenantId`**
    - *Type and Technical Role:* `@fulfillmenttools/n8n-nodes-fulfillmenttools.fulfillmenttools` (API Action).
    - *Configuration Choices:* Resource: `order`, Operation: `getAll` with a filter matching `tenantOrderId`.
    - *Key Expressions/Variables:* `={{ $('Handoverjob Created').item.json.payload.tenantOrderId }}`
    - *Input/Output Connections:* Inputs: `Check If Data is Valid` | Outputs: `Get Order by Id`.
    - *Version Requirements:* v1.
    - *Edge Cases/Failure Types:* Network timeouts, API rate limits, or non-existent tenant order IDs returning empty arrays.

  - **`Get Order by Id`**
    - *Type and Technical Role:* `@fulfillmenttools/n8n-nodes-fulfillmenttools.fulfillmenttools` (API Action).
    - *Configuration Choices:* Resource: `order`, Operation: `get` using the internal ID returned by the previous search node.
    - *Key Expressions/Variables:* `={{ $json.id }}`
    - *Input/Output Connections:* Inputs: `Get Order by tenantId` | Outputs: `Check for valid email address`.
    - *Version Requirements:* v1.
    - *Edge Cases/Failure Types:* API resource lookup errors if the internal ID resolution fails.

  - **`Check for valid email address`**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Conditional Router).
    - *Configuration Choices:* Validates that the customer email address exists and is not empty.
    - *Key Expressions/Variables:* 
      - Left Value: `={{ $json.consumer.addresses[0].email }}`
      - Operators: `exists` and `notEmpty` (combined with `and`).
    - *Input/Output Connections:* Inputs: `Get Order by Id` | Outputs (True): `Get pickup Facility` | Outputs (False): `No Operation, do nothing`.
    - *Version Requirements:* v2.3.
    - *Edge Cases/Failure Types:* Accessing array index `[0]` on an empty `addresses` array causes an undefined property error.

---

#### 2.3 Facility Lookup and Language Detection
- **Overview:** This block fetches physical store information for the pickup location and routes the workflow based on the customer's specified shop language.
- **Nodes Involved:**
  - `Get pickup Facility`
  - `Select Language`

- **Node Details:**
  - **`Get pickup Facility`**
    - *Type and Technical Role:* `@fulfillmenttools/n8n-nodes-fulfillmenttools.fulfillmenttools` (API Action).
    - *Configuration Choices:* Resource: `facility`, Operation: `get` using the facility reference from the trigger payload.
    - *Key Expressions/Variables:* `={{ $('Handoverjob Created').item.json.payload.facilityRef }}`
    - *Input/Output Connections:* Inputs: `Check for valid email address` | Outputs: `Select Language`.
    - *Version Requirements:* v1.
    - *Edge Cases/Failure Types:* Invalid facility references or missing fulfillmenttools configuration parameters.

  - **`Select Language`**
    - *Type and Technical Role:* `n8n-nodes-base.switch` (Multi-path Router).
    - *Configuration Choices:* Evaluates the custom attribute `shopLanguage` from the trigger payload and splits output routing accordingly.
    - *Key Expressions/Variables:* 
      - Rule 1 (`en_US`): `={{ $('Handoverjob Created').item.json.payload.customAttributes.shopLanguage }}` equals `en_US`.
      - Rule 2 (`de_DE`): `={{ $('Handoverjob Created').item.json.payload.customAttributes.shopLanguage }}` equals `de_DE`.
    - *Input/Output Connections:* Inputs: `Get pickup Facility` | Outputs: Output 0 (`en_US`) connects to `Send pickup email english`, Output 1 (`de_DE`) connects to `Send pickup email german`.
    - *Version Requirements:* v3.4.
    - *Edge Cases/Failure Types:* Unhandled language codes will fall through without matching any route unless a fallback rule is configured.

---

#### 2.4 Multilingual Email Dispatch
- **Overview:** Compiles dynamic HTML templates incorporating order details, line items, pricing, and store location data, then sends the email via Gmail.
- **Nodes Involved:**
  - `Send pickup email english`
  - `Send pickup email german`

- **Node Details:**
  - **`Send pickup email english`**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (Email Dispatch).
    - *Configuration Choices:* Sends an HTML-formatted message using Gmail OAuth2 credentials. Subject includes the dynamic tenant order ID.
    - *Key Expressions/Variables:*
      - Recipient: `={{ $('Get Order by Id').item.json.consumer.addresses[0].email }}`
      - Subject: `=Your Order {{ $('Handoverjob Created').item.json.payload.tenantOrderId }} at fulfillmenttools Shop is ready for pickup`
      - Body: Embedded HTML template utilizing JavaScript map and Intl number formatting functions to iterate over `handoverJobLineItems`.
    - *Input/Output Connections:* Inputs: `Select Language` (Output 0) | Outputs: None (Terminal).
    - *Version Requirements:* v2.2.
    - *Edge Cases/Failure Types:* OAuth2 token expiration, exceeded sending quotas, or malformed HTML template expressions.

  - **`Send pickup email german`**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (Email Dispatch).
    - *Configuration Choices:* Sends a German localized HTML-formatted message using Gmail OAuth2 credentials.
    - *Key Expressions/Variables:*
      - Recipient: `={{ $('Get Order by Id').item.json.consumer.addresses[0].email }}`
      - Subject: `=Ihre Bestellung {{ $('Handoverjob Created').item.json.payload.tenantOrderId }} bei fulfillmenttools Shop wird geliefert`
      - Body: Localized German HTML template utilizing JavaScript array mapping and `de-DE` currency formatting.
    - *Input/Output Connections:* Inputs: `Select Language` (Output 1) | Outputs: None (Terminal).
    - *Version Requirements:* v2.2.
    - *Edge Cases/Failure Types:* OAuth2 token revocation, missing permissions in Google Cloud Console.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Handoverjob Created` | `@fulfillmenttools/n8n-nodes-fulfillmenttools.fulfillmenttoolsTrigger` | Triggers workflow on new handover jobs | None | `Check If Data is Valid` | # Send an email to a customer when his Order in fulfillmenttools is ready for pickup...<br>## Get and check Data<br>The fulfillmenttools Trigger will deliver all Handoverjobs that have been created and therefore are now ready for pickup for the Customer... |
| `Check If Data is Valid` | `n8n-nodes-base.if` | Validates sales source and channel | `Handoverjob Created` | `Get Order by tenantId`, `No Operation, do nothing` | ## Get and check Data<br>The fulfillmenttools Trigger will deliver all Handoverjobs that have been created and therefore are now ready for pickup for the Customer... |
| `No Operation, do nothing` | `n8n-nodes-base.noOp` | Terminal node for invalid filter paths | `Check If Data is Valid`, `Check for valid email address` | None | # Send an email to a customer when his Order in fulfillmenttools is ready for pickup... |
| `Get Order by tenantId` | `@fulfillmenttools/n8n-nodes-fulfillmenttools.fulfillmenttools` | Searches order by tenant order ID | `Check If Data is Valid` | `Get Order by Id` | ## Collect Order Data and check for email address<br>The Workflow fetches the original Order of this customer from fulfillmenttools. In this data the email address of the customer is saved and gets checked in the last step. |
| `Get Order by Id` | `@fulfillmenttools/n8n-nodes-fulfillmenttools.fulfillmenttools` | Fetches full order details | `Get Order by tenantId` | `Check for valid email address` | ## Collect Order Data and check for email address<br>The Workflow fetches the original Order of this customer from fulfillmenttools. In this data the email address of the customer is saved and gets checked in the last step. |
| `Check for valid email address` | `n8n-nodes-base.if` | Verifies customer email existence | `Get Order by Id` | `Get pickup Facility`, `No Operation, do nothing` | ## Collect Order Data and check for email address<br>The Workflow fetches the original Order of this customer from fulfillmenttools. In this data the email address of the customer is saved and gets checked in the last step. |
| `Get pickup Facility` | `@fulfillmenttools/n8n-nodes-fulfillmenttools.fulfillmenttools` | Retrieves pickup location details | `Check for valid email address` | `Select Language` | ## Collect pickup Facility Data and check for Language<br>To have all the data needed for our email Template this step collects the selected pickup Facility for this Order and checks which language did the customer use inside our Shop... |
| `Select Language` | `n8n-nodes-base.switch` | Routes flow by customer shop language | `Get pickup Facility` | `Send pickup email english`, `Send pickup email german` | ## Collect pickup Facility Data and check for Language<br>To have all the data needed for our email Template this step collects the selected pickup Facility for this Order and checks which language did the customer use inside our Shop... |
| `Send pickup email english` | `n8n-nodes-base.gmail` | Sends English pickup notification email | `Select Language` | None | ## Send emails to Customer<br>The Workflow connects to gmail and sends the email in the correct language to the customer. Here a HTML message is used that relies on data from the Workflow... |
| `Send pickup email german` | `n8n-nodes-base.gmail` | Sends German pickup notification email | `Select Language` | None | ## Send emails to Customer<br>The Workflow connects to gmail and sends the email in the correct language to the customer. Here a HTML message is used that relies on data from the Workflow... |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**
   - Add a **fulfillmenttools Trigger** node named `Handoverjob Created`.
   - Set the event parameter to `HANDOVERJOB_CREATED`.
   - Configure the subscription name to `n8n Shop Pickup ready email`.
   - Connect valid `fulfillmenttoolsApi` credentials.

2. **Add Data Validation:**
   - Add an **If** node named `Check If Data is Valid`.
   - Set condition 1: `{{ $json.payload.customAttributes.salesSource }}` equals `FFT_Shop`.
   - Set condition 2: `{{ $json.payload.channel }}` equals `COLLECT`.
   - Connect the `true` output of `Check If Data is Valid` to the next step.

3. **Configure Fallback/Termination Node:**
   - Add a **No Operation, do nothing** node named `No Operation, do nothing`.
   - Connect the `false` outputs of both `Check If Data is Valid` and `Check for valid email address` to this node.

4. **Fetch Order Information:**
   - Add a **fulfillmenttools** node named `Get Order by tenantId`.
   - Set resource to `order` and operation to `getAll`.
   - Add a filter for `tenantOrderId` set to `={{ $('Handoverjob Created').item.json.payload.tenantOrderId }}`.
   - Connect `Check If Data is Valid` (true) to `Get Order by tenantId`.

5. **Retrieve Detailed Order Record:**
   - Add a second **fulfillmenttools** node named `Get Order by Id`.
   - Set resource to `order` and operation to `get`.
   - Set `orderId` to `={{ $json.id }}`.
   - Connect `Get Order by tenantId` to `Get Order by Id`.

6. **Validate Email Address:**
   - Add an **If** node named `Check for valid email address`.
   - Set condition to verify that `={{ $json.consumer.addresses[0].email }}` exists and is not empty.
   - Connect `Get Order by Id` to this node.

7. **Retrieve Pickup Facility Data:**
   - Add a **fulfillmenttools** node named `Get pickup Facility`.
   - Set resource to `facility` and operation to `get`.
   - Set `facilityId` to `={{ $('Handoverjob Created').item.json.payload.facilityRef }}`.
   - Connect the `true` output of `Check for valid email address` to this node.

8. **Implement Language Routing:**
   - Add a **Switch** node named `Select Language`.
   - Create rule output `en_US` where `={{ $('Handoverjob Created').item.json.payload.customAttributes.shopLanguage }}` equals `en_US`.
   - Create rule output `de_DE` where `={{ $('Handoverjob Created').item.json.payload.customAttributes.shopLanguage }}` equals `de_DE`.
   - Connect `Get pickup Facility` to `Select Language`.

9. **Configure English Email Dispatch:**
   - Add a **Gmail** node named `Send pickup email english`.
   - Configure credentials using a valid `gmailOAuth2` account.
   - Set recipient to `={{ $('Get Order by Id').item.json.consumer.addresses[0].email }}`.
   - Set subject and HTML message body using localized English templates iterating over line items.
   - Connect `Select Language` output `en_US` to this node.

10. **Configure German Email Dispatch:**
    - Add a **Gmail** node named `Send pickup email german`.
    - Configure credentials using a valid `gmailOAuth2` account.
    - Set recipient to `={{ $('Get Order by Id').item.json.consumer.addresses[0].email }}`.
    - Set subject and HTML message body using localized German templates iterating over line items.
    - Connect `Select Language` output `de_DE` to this node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| fulfillmenttools Community Node documentation and setup instructions | [fulfillmenttools Node in n8n community](https://docs.n8n.io/integrations/community-nodes/) |
| Google Fonts Manrope and Inter typography styling used in email templates | [Google Fonts Library](https://fonts.google.com/) |
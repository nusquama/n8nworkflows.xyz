Generate AI text and images with Claude and Stripe metered billing

https://n8nworkflows.xyz/workflows/generate-ai-text-and-images-with-claude-and-stripe-metered-billing-19670


# Generate AI text and images with Claude and Stripe metered billing

### 1. Workflow Overview

This workflow acts as the backend service for an AI-powered Software-as-a-Service (SaaS) application. It exposes a synchronous HTTP webhook that processes generation requests by authenticating users via Stripe API keys, verifying active subscriptions and metered usage quotas, generating text (via Anthropic Claude) or images (via a custom external API), recording the metered usage increment in Stripe, and returning a JSON payload to the caller.

The workflow logic is categorized into five functional blocks:
- **1.1 Input Reception & Configuration:** Receives incoming POST requests and normalizes request payloads alongside configuration mappings (models, limits).
- **1.2 Authentication & Customer Lookup:** Queries Stripe to locate customers using metadata-stored API keys and handles unauthorized requests.
- **1.3 Subscription & Entitlement Check:** Fetches active subscriptions and usage summaries from Stripe to evaluate current quota status against predefined plan limits.
- **1.4 AI Content Generation:** Evaluates requested content type and routes execution to either Anthropic Claude (for text) or an external image-generation endpoint, consolidating results via a merge node.
- **1.5 Usage Recording, Billing & Response:** Increments usage records in Stripe and returns synchronous responses for both successful generations and blocked/unauthorized scenarios.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Configuration
- **Overview:** Captures incoming HTTP POST requests containing user credentials and parameters, centralizing execution variables and plan limit definitions.
- **Nodes Involved:** 
  - `Webhook - Generation Request`
  - `Set - Config`

##### Node Details:
- **Webhook - Generation Request**
  - *Type & Technical Role:* `n8n-nodes-base.webhook` (Trigger node). Exposes a production/test webhook endpoint expecting `POST` requests at `/ai-saas/generate`.
  - *Configuration Choices:* Response mode set to use a downstream response node (`responseNode`).
  - *Key Expressions/Variables:* None.
  - *Connections:* Input: None (Trigger); Output: `Set - Config`.
  - *Edge Cases/Failure Types:* Network timeouts or incorrect HTTP methods (returns default n8n error handling).

- **Set - Config**
  - *Type & Technical Role:* `n8n-nodes-base.set` (Data transformation node). Centralizes global configuration variables, default parameters, model assignments, and a JSON dictionary mapping Stripe price IDs to monthly limits.
  - *Configuration Choices:* Maps incoming payload properties (`apiKey`, `contentType`, `prompt`, `params`) and establishes fallback parameters (`anthropicModel`, `imageGenEndpoint`, `stripeUsagePriceIdFallback`, `planLimits`).
  - *Key Expressions/Variables:* `={{ $json.body?.apiKey }}`, `={{ $json.body?.contentType || 'text' }}`, `={{ $json.body?.prompt }}`, `={{ $json.body?.params || {} }}`.
  - *Connections:* Input: `Webhook - Generation Request`; Output: `HTTP Request - Stripe: Find Customer By API Key`.
  - *Edge Cases/Failure Types:* Expression evaluation failures if body is missing or malformed.

---

#### 2.2 Authentication & Customer Lookup
- **Overview:** Queries the Stripe Customer Search API using the incoming API key stored in customer metadata, validating whether a corresponding account exists.
- **Nodes Involved:** 
  - `HTTP Request - Stripe: Find Customer By API Key`
  - `JS - Resolve Customer`
  - `IF - Customer Found`
  - `Respond - Unauthorized`

##### Node Details:
- **HTTP Request - Stripe: Find Customer By API Key**
  - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (Integration node). Queries Stripe's customer search endpoint.
  - *Configuration Choices:* Uses predefined Stripe API credentials. Appends query parameters to search metadata keys.
  - *Key Expressions/Variables:* `=metadata['apiKey']:'{{ $json.apiKey }}'`
  - *Connections:* Input: `Set - Config`; Output: `JS - Resolve Customer`.
  - *Edge Cases/Failure Types:* Rate limiting from Stripe API, invalid credentials (`continueOnFail` enabled).

- **JS - Resolve Customer**
  - *Type & Technical Role:* `n8n-nodes-base.code` (Data transformation node). Extracts the first matching customer object from Stripe's search array and merges configuration state.
  - *Configuration Choices:* JavaScript execution to parse response arrays safely.
  - *Key Expressions/Variables:* Extracts `customer.id`, `customer.email`, and `customer.metadata`.
  - *Connections:* Input: `HTTP Request - Stripe: Find Customer By API Key`; Output: `IF - Customer Found`.

- **IF - Customer Found**
  - *Type & Technical Role:* `n8n-nodes-base.if` (Flow control node). Routes execution based on whether a valid customer was located.
  - *Configuration Choices:* Evaluates boolean value of `customerFound`.
  - *Key Expressions/Variables:* `={{ $json.customerFound === true }}`
  - *Connections:* Input: `JS - Resolve Customer`; Outputs: `HTTP Request - Stripe: Get Active Subscription` (True branch), `Respond - Unauthorized` (False branch).

- **Respond - Unauthorized**
  - *Type & Technical Role:* `n8n-nodes-base.respondToWebhook` (Response node). Terminates execution for invalid API keys.
  - *Configuration Choices:* Sets HTTP response code to `401` and returns a JSON error body.
  - *Key Expressions/Variables:* `={{ JSON.stringify({ success: false, error: 'invalid_api_key' }) }}`
  - *Connections:* Input: `IF - Customer Found` (False branch); Output: None.

---

#### 2.3 Subscription & Entitlement Check
- **Overview:** Fetches active subscriptions, parses active metered items, pulls current usage summaries, and checks quotas against plan limits.
- **Nodes Involved:**
  - `HTTP Request - Stripe: Get Active Subscription`
  - `JS - Parse Subscription`
  - `HTTP Request - Stripe: Get Current Period Usage`
  - `JS - Check Entitlement`
  - `IF - Entitlement OK`
  - `Respond - Entitlement Blocked`

##### Node Details:
- **HTTP Request - Stripe: Get Active Subscription**
  - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (Integration node). Fetches active subscriptions linked to the customer ID.
  - *Configuration Choices:* Queries Stripe subscriptions endpoint filtering by customer and active status.
  - *Key Expressions/Variables:* `={{ $json.customerId }}`
  - *Connections:* Input: `IF - Customer Found` (True branch); Output: `JS - Parse Subscription`.

- **JS - Parse Subscription**
  - *Type & Technical Role:* `n8n-nodes-base.code` (Data transformation node). Extracts subscription items and price IDs from the subscription payload.
  - *Key Expressions/Variables:* Reads `subscription.items.data[0]`.
  - *Connections:* Input: `HTTP Request - Stripe: Get Active Subscription`; Output: `HTTP Request - Stripe: Get Current Period Usage`.

- **HTTP Request - Stripe: Get Current Period Usage**
  - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (Integration node). Pulls usage record summaries for the current billing period from the active subscription item.
  - *Key Expressions/Variables:* Dynamic URL using `={{ $json.subscriptionItemId }}`.
  - *Connections:* Input: `JS - Parse Subscription`; Output: `JS - Check Entitlement`.

- **JS - Check Entitlement**
  - *Type & Technical Role:* `n8n-nodes-base.code` (Data transformation node). Compares current usage totals against plan limits defined in the configuration map.
  - *Key Expressions/Variables:* `ctx.planLimits?.[ctx.subscriptionPriceId] ?? 100`.
  - *Connections:* Input: `HTTP Request - Stripe: Get Current Period Usage`; Output: `IF - Entitlement OK`.

- **IF - Entitlement OK**
  - *Type & Technical Role:* `n8n-nodes-base.if` (Flow control node). Evaluates whether the user is authorized to proceed with content generation based on usage thresholds and subscription status.
  - *Key Expressions/Variables:* `={{ $json.entitlementAllowed === true }}`
  - *Connections:* Input: `JS - Check Entitlement`; Outputs: `Switch - Content Type` (True branch), `Respond - Entitlement Blocked` (False branch).

- **Respond - Entitlement Blocked**
  - *Type & Technical Role:* `n8n-nodes-base.respondToWebhook` (Response node). Returns a `402 Payment Required` response when usage limits are exceeded or subscriptions are inactive.
  - *Key Expressions/Variables:* Returns JSON with reason codes, current usage, and usage limits.
  - *Connections:* Input: `IF - Entitlement OK` (False branch); Output: None.

---

#### 2.4 AI Content Generation
- **Overview:** Routes requests based on content type to either Anthropic Claude for text generation or an external endpoint for image generation, then unifies the results.
- **Nodes Involved:**
  - `Switch - Content Type`
  - `HTTP Request - Claude: Generate Text Content`
  - `JS - Parse Text Result`
  - `HTTP Request - Generate Image Content`
  - `JS - Parse Image Result`
  - `Merge - Combine Generation Results`

##### Node Details:
- **Switch - Content Type**
  - *Type & Technical Role:* `n8n-nodes-base.switch` (Flow control node). Directs execution based on whether `contentType` equals `text` or `image`.
  - *Key Expressions/Variables:* `={{ $json.contentType }}`
  - *Connections:* Input: `IF - Entitlement OK` (True branch); Outputs: `HTTP Request - Claude: Generate Text Content` (Route 0), `HTTP Request - Generate Image Content` (Route 1).

- **HTTP Request - Claude: Generate Text Content**
  - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (Integration node). Calls the Anthropic Messages API.
  - *Configuration Choices:* Uses HTTP Header Auth credentials (`x-api-key`), JSON body payload with model name, max tokens, and user prompt.
  - *Key Expressions/Variables:* `={{ JSON.stringify({ model: $json.anthropicModel, max_tokens: 2000, messages: [ { role: 'user', content: $json.prompt } ] }) }}`
  - *Connections:* Input: `Switch - Content Type`; Output: `JS - Parse Text Result`.

- **JS - Parse Text Result**
  - *Type & Technical Role:* `n8n-nodes-base.code` (Data transformation node). Extracts the text response block from Anthropic's response structure.
  - *Connections:* Input: `HTTP Request - Claude: Generate Text Content`; Output: `Merge - Combine Generation Results`.

- **HTTP Request - Generate Image Content**
  - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (Integration node). Calls a configured external image generation endpoint.
  - *Configuration Choices:* Uses HTTP Header Auth, JSON body containing prompt and generation parameters.
  - *Key Expressions/Variables:* `={{ $json.imageGenEndpoint }}`
  - *Connections:* Input: `Switch - Content Type`; Output: `JS - Parse Image Result`.

- **JS - Parse Image Result**
  - *Type & Technical Role:* `n8n-nodes-base.code` (Data transformation node). Parses image URLs from various potential JSON response structures.
  - *Connections:* Input: `HTTP Request - Generate Image Content`; Output: `Merge - Combine Generation Results`.

- **Merge - Combine Generation Results**
  - *Type & Technical Role:* `n8n-nodes-base.merge` (Data combination node). Merges execution paths from text or image generation branches using "chooseBranch" mode.
  - *Connections:* Inputs: `JS - Parse Text Result` and `JS - Parse Image Result`; Output: `HTTP Request - Stripe: Record Usage`.

---

#### 2.5 Usage Recording, Billing & Response
- **Overview:** Increments the usage record on the Stripe metered subscription item and returns a successful JSON payload to the client.
- **Nodes Involved:**
  - `HTTP Request - Stripe: Record Usage`
  - `JS - Build Success Response`
  - `Respond - Success`

##### Node Details:
- **HTTP Request - Stripe: Record Usage**
  - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (Integration node). Posts usage increments to Stripe.
  - *Configuration Choices:* Uses form-urlencoded content type with `quantity: 1`, `action: increment`, and a Unix timestamp.
  - *Key Expressions/Variables:* Dynamic URL with `subscriptionItemId`, `={{ Math.floor(Date.now() / 1000) }}`.
  - *Connections:* Input: `Merge - Combine Generation Results`; Output: `JS - Build Success Response`.

- **JS - Build Success Response**
  - *Type & Technical Role:* `n8n-nodes-base.code` (Data transformation node). Shapes the final success JSON payload containing generated content and updated usage metrics.
  - *Connections:* Input: `HTTP Request - Stripe: Record Usage`; Output: `Respond - Success`.

- **Respond - Success**
  - *Type & Technical Role:* `n8n-nodes-base.respondToWebhook` (Response node). Returns a successful `200 OK` HTTP response to the caller.
  - *Configuration Choices:* Responds with JSON using `={{ $json }}`.
  - *Connections:* Input: `JS - Build Success Response`; Output: None.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Sticky Note - Overview` | `n8n-nodes-base.stickyNote` | Documentation block explaining workflow purpose and setup instructions. | None | None | ## AI SaaS Backend (Content Generation + Stripe Billing)... |
| `Sticky Note - Trigger & Auth` | `n8n-nodes-base.stickyNote` | Documentation block covering triggers, configuration, and API key lookup logic. | None | None | ## 1. Trigger, Config & Auth... |
| `Sticky Note - Subscription & Usage Check` | `n8n-nodes-base.stickyNote` | Documentation block covering subscription verification and entitlement checks. | None | None | ## 2. Subscription & Entitlement Check... |
| `Sticky Note - AI Content Generation` | `n8n-nodes-base.stickyNote` | Documentation block outlining text and image generation routing. | None | None | ## 3. AI Content Generation... |
| `Sticky Note - Usage Recording & Response` | `n8n-nodes-base.stickyNote` | Documentation block covering metered usage reporting and response formatting. | None | None | ## 4. Usage Recording, Billing & Response... |
| `Webhook - Generation Request` | `n8n-nodes-base.webhook` | Receives client POST requests at `/ai-saas/generate`. | None | `Set - Config` | ## 1. Trigger, Config & Auth |
| `Set - Config` | `n8n-nodes-base.set` | Centralizes runtime variables, model versions, and plan usage limits. | `Webhook - Generation Request` | `HTTP Request - Stripe: Find Customer By API Key` | ## 1. Trigger, Config & Auth |
| `HTTP Request - Stripe: Find Customer By API Key` | `n8n-nodes-base.httpRequest` | Searches Stripe customers for metadata matching the incoming API key. | `Set - Config` | `JS - Resolve Customer` | ## 1. Trigger, Config & Auth |
| `JS - Resolve Customer` | `n8n-nodes-base.code` | Parses customer search results and normalizes output variables. | `HTTP Request - Stripe: Find Customer By API Key` | `IF - Customer Found` | ## 1. Trigger, Config & Auth |
| `IF - Customer Found` | `n8n-nodes-base.if` | Routes valid customers to subscription check; invalid keys to unauthorized response. | `JS - Resolve Customer` | `HTTP Request - Stripe: Get Active Subscription`, `Respond - Unauthorized` | ## 1. Trigger, Config & Auth |
| `HTTP Request - Stripe: Get Active Subscription` | `n8n-nodes-base.httpRequest` | Fetches active subscriptions and metered subscription items for the customer. | `IF - Customer Found` | `JS - Parse Subscription` | ## 2. Subscription & Entitlement Check |
| `JS - Parse Subscription` | `n8n-nodes-base.code` | Extracts subscription ID and price item metadata. | `HTTP Request - Stripe: Get Active Subscription` | `HTTP Request - Stripe: Get Current Period Usage` | ## 2. Subscription & Entitlement Check |
| `HTTP Request - Stripe: Get Current Period Usage` | `n8n-nodes-base.httpRequest` | Pulls usage record summaries for the current billing cycle. | `JS - Parse Subscription` | `JS - Check Entitlement` | ## 2. Subscription & Entitlement Check |
| `JS - Check Entitlement` | `n8n-nodes-base.code` | Compares current usage against monthly plan limits. | `HTTP Request - Stripe: Get Current Period Usage` | `IF - Entitlement OK` | ## 2. Subscription & Entitlement Check |
| `IF - Entitlement OK` | `n8n-nodes-base.if` | Routes authorized requests to content generation; blocked requests to error response. | `JS - Check Entitlement` | `Switch - Content Type`, `Respond - Entitlement Blocked` | ## 2. Subscription & Entitlement Check |
| `Switch - Content Type` | `n8n-nodes-base.switch` | Directs execution to text or image generation based on request parameters. | `IF - Entitlement OK` | `HTTP Request - Claude: Generate Text Content`, `HTTP Request - Generate Image Content` | ## 3. AI Content Generation |
| `HTTP Request - Claude: Generate Text Content` | `n8n-nodes-base.httpRequest` | Calls the Anthropic Claude API to generate text responses. | `Switch - Content Type` | `JS - Parse Text Result` | ## 3. AI Content Generation |
| `JS - Parse Text Result` | `n8n-nodes-base.code` | Extracts text blocks from the Anthropic API response. | `HTTP Request - Claude: Generate Text Content` | `Merge - Combine Generation Results` | ## 3. AI Content Generation |
| `HTTP Request - Generate Image Content` | `n8n-nodes-base.httpRequest` | Calls an external image generation endpoint. | `Switch - Content Type` | `JS - Parse Image Result` | ## 3. AI Content Generation |
| `JS - Parse Image Result` | `n8n-nodes-base.code` | Parses image URLs from external generation responses. | `HTTP Request - Generate Image Content` | `Merge - Combine Generation Results` | ## 3. AI Content Generation |
| `Merge - Combine Generation Results` | `n8n-nodes-base.merge` | Unifies text and image generation branches. | `JS - Parse Text Result`, `JS - Parse Image Result` | `HTTP Request - Stripe: Record Usage` | ## 3. AI Content Generation |
| `HTTP Request - Stripe: Record Usage` | `n8n-nodes-base.httpRequest` | Posts a usage record increment to the customer's Stripe subscription item. | `Merge - Combine Generation Results` | `JS - Build Success Response` | ## 4. Usage Recording, Billing & Response |
| `JS - Build Success Response` | `n8n-nodes-base.code` | Formats the final success response payload. | `HTTP Request - Stripe: Record Usage` | `Respond - Success` | ## 4. Usage Recording, Billing & Response |
| `Respond - Success` | `n8n-nodes-base.respondToWebhook` | Returns a `200 OK` response with generated content and usage statistics. | `JS - Build Success Response` | None | ## 4. Usage Recording, Billing & Response |
| `Respond - Unauthorized` | `n8n-nodes-base.respondToWebhook` | Returns a `401 Unauthorized` HTTP response for invalid API keys. | `IF - Customer Found` | None | ## 1. Trigger, Config & Auth |
| `Respond - Entitlement Blocked` | `n8n-nodes-base.respondToWebhook` | Returns a `402 Payment Required` HTTP response when usage limits are exceeded. | `IF - Entitlement OK` | None | ## 2. Subscription & Entitlement Check |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Trigger:** Add a **Webhook** node named `Webhook - Generation Request`. Set the HTTP Method to `POST`, path to `ai-saas/generate`, and response mode to `responseNode`.
2. **Configure Global Variables:** Add a **Set** node named `Set - Config`. Assign string and object values for `apiKey`, `contentType`, `prompt`, `generationParams`, `anthropicModel` (`claude-sonnet-4-6`), `imageGenEndpoint`, `stripeUsagePriceIdFallback`, and a `planLimits` JSON object mapping price IDs to monthly usage quotas. Connect `Webhook - Generation Request` to this node.
3. **Lookup Stripe Customer:** Add an **HTTP Request** node named `HTTP Request - Stripe: Find Customer By API Key`. Configure authentication using Stripe API credentials (`stripeApi`). Set URL to `https://api.stripe.com/v1/customers/search` with query parameter `query` set to `=metadata['apiKey']:'{{ $json.apiKey }}'`. Enable `continueOnFail`. Connect `Set - Config` to this node.
4. **Resolve Customer Data:** Add a **Code** node named `JS - Resolve Customer` to parse customer search arrays and pass metadata downstream. Connect from `HTTP Request - Stripe: Find Customer By API Key`.
5. **Add Customer Validation Branch:** Add an **If** node named `IF - Customer Found`. Configure condition to check if `customerFound === true`. 
   - True branch connects to subscription retrieval.
   - False branch connects to a **Respond to Webhook** node named `Respond - Unauthorized` configured with response code `401` returning `{ success: false, error: 'invalid_api_key' }`.
6. **Fetch Active Subscriptions:** Add an **HTTP Request** node named `HTTP Request - Stripe: Get Active Subscription`. Set URL to `https://api.stripe.com/v1/subscriptions` with query parameters for `customer` (`{{ $json.customerId }}`), `status` (`active`), and `limit` (`1`). Enable `continueOnFail`. Connect to the True output of `IF - Customer Found`.
7. **Parse Subscription Data:** Add a **Code** node named `JS - Parse Subscription` to extract subscription items and price IDs. Connect from `HTTP Request - Stripe: Get Active Subscription`.
8. **Fetch Current Usage:** Add an **HTTP Request** node named `HTTP Request - Stripe: Get Current Period Usage`. Set URL to `=https://api.stripe.com/v1/subscription_items/{{ $json.subscriptionItemId }}/usage_record_summaries` with a limit of `1`. Enable `continueOnFail`. Connect from `JS - Parse Subscription`.
9. **Evaluate Entitlement:** Add a **Code** node named `JS - Check Entitlement` to compare current period usage against the plan limit. Connect from `HTTP Request - Stripe: Get Current Period Usage`.
10. **Add Entitlement Validation Branch:** Add an **If** node named `IF - Entitlement OK`. Configure condition to check if `entitlementAllowed === true`.
    - True branch connects to content routing.
    - False branch connects to a **Respond to Webhook** node named `Respond - Entitlement Blocked` configured with response code `402` returning error details.
11. **Route Content Generation:** Add a **Switch** node named `Switch - Content Type`. Create two rules checking if `contentType` equals `text` or `image`. Connect from the True output of `IF - Entitlement OK`.
12. **Configure Text Generation:** 
    - Add an **HTTP Request** node named `HTTP Request - Claude: Generate Text Content` connected to Route 0 of the switch. Set method to `POST`, URL to `https://api.anthropic.com/v1/messages`, and authenticate using Predefined Header Auth (`httpHeaderAuth`) with headers `anthropic-version: 2023-06-01` and `content-type: application/json`. Set JSON body containing the model and prompt.
    - Add a **Code** node named `JS - Parse Text Result` to extract text blocks. Connect from the HTTP request node.
13. **Configure Image Generation:** 
    - Add an **HTTP Request** node named `HTTP Request - Generate Image Content` connected to Route 1 of the switch. Set method to `POST`, URL to `={{ $json.imageGenEndpoint }}`, authenticate via header auth, and pass prompt parameters in the JSON body.
    - Add a **Code** node named `JS - Parse Image Result` to extract image URLs. Connect from the HTTP request node.
14. **Merge Generation Results:** Add a **Merge** node named `Merge - Combine Generation Results` set to `chooseBranch` mode. Connect both `JS - Parse Text Result` and `JS - Parse Image Result` into inputs 0 and 1.
15. **Record Stripe Usage:** Add an **HTTP Request** node named `HTTP Request - Stripe: Record Usage`. Set method to `POST`, URL to `=https://api.stripe.com/v1/subscription_items/{{ $json.subscriptionItemId }}/usage_records`, content type to `form-urlencoded`, and body parameters for `quantity` (`1`), `action` (`increment`), and `timestamp` (`={{ Math.floor(Date.now() / 1000) }}`). Connect from `Merge - Combine Generation Results`.
16. **Format Success Response:** Add a **Code** node named `JS - Build Success Response` to structure the final payload. Connect from `HTTP Request - Stripe: Record Usage`.
17. **Send Success Response:** Add a **Respond to Webhook** node named `Respond - Success` configured to respond with JSON using `={{ $json }}`. Connect from `JS - Build Success Response`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Stripe API documentation for Metered Billing and Usage Records | [Stripe Billing Documentation](https://stripe.com/docs/billing/subscriptions/metered-billing) |
| Anthropic Claude Messages API reference | [Anthropic API Documentation](https://docs.anthropic.com/claude/reference/messages_post) |
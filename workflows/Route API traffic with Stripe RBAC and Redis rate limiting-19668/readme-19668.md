Route API traffic with Stripe RBAC and Redis rate limiting

https://n8nworkflows.xyz/workflows/route-api-traffic-with-stripe-rbac-and-redis-rate-limiting-19668


# Route API traffic with Stripe RBAC and Redis rate limiting

### 1. Workflow Overview

This workflow acts as an API gateway front-door for backend services, providing security, authorization, and rate-limiting enforcement before any internal logic executes. It intercepts incoming HTTP POST requests, validates the provided API key against Stripe customer metadata, enforces Role-Based Access Control (RBAC), limits request frequency using Redis, and routes valid requests to corresponding internal webhooks.

The logical execution divides into four key operational blocks:
- **1.1 Input Reception & Configuration:** Ingests the raw API request and injects global configuration maps (routing rules, rate limits, windows).
- **1.2 Authentication & Role Resolution:** Queries Stripe to authenticate the API key, extracts user metadata, and bifurcates unauthorized traffic into a 401 response.
- **1.3 RBAC & Rate Limiting:** Validates that the user's role is permitted to access the requested route, increments and checks Redis sliding-window counters, and intercepts disallowed or over-limit requests with 403 or 429 errors respectively.
- **1.4 Internal Dispatch & Client Response:** Maps valid requests to internal n8n webhook URLs, forwards context payload data, appends tracking headers, and responds to the origin client.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Configuration
This block receives the inbound API payload and establishes operational parameters including permissions maps, rate limit allocations, and internal routing endpoints.

- **Webhook - Gateway Entry**
  - *Type & Technical Role:* `n8n-nodes-base.webhook` (Trigger)
  - *Configuration:* Listens on path `gateway` for `POST` methods using response mode `responseNode`.
  - *Key Expressions:* None.
  - *Connections:* Input: None (Trigger); Output: `Set - Gateway Config`.
  - *Edge Cases / Failures:* Invalid paths or incorrect HTTP methods return standard n8n gateway errors.
- **Set - Gateway Config**
  - *Type & Technical Role:* `n8n-nodes-base.set` (Data Transformation)
  - *Configuration:* Assigns standard gateway configuration parameters (`apiKey`, `targetRoute`, `method`, `payload`, `rateLimitWindowSeconds`, `rateLimits`, `routePermissions`, `internalWorkflowMap`).
  - *Key Expressions:* 
    - `apiKey`: `={{ $json.headers?.['x-api-key'] || $json.body?.apiKey }}`
    - `targetRoute`: `={{ $json.body?.route || $json.query?.route }}`
    - `method`: `={{ $json.body?.method || 'POST' }}`
    - `payload`: `={{ $json.body?.payload || {} }}`
  - *Connections:* Input: `Webhook - Gateway Entry`; Output: `HTTP Request - Stripe: Find Customer By API Key`.

#### 2.2 Authentication & Role Resolution
This block queries the Stripe API to verify the API key, validates metadata structures, defaults missing roles, and separates valid from invalid consumers.

- **HTTP Request - Stripe: Find Customer By API Key**
  - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (Integration)
  - *Configuration:* Calls Stripe customers search endpoint (`https://api.stripe.com/v1/customers/search`) using predefined Stripe credentials and query parameters.
  - *Key Expressions:* Query string parameter `query` = `=metadata['apiKey']:'{{ $json.apiKey }}'`
  - *Connections:* Input: `Set - Gateway Config`; Output: `JS - Resolve Role`.
  - *Edge Cases / Failures:* Network failures or misconfigured Stripe keys. `continueOnFail` is enabled to gracefully pass errors down the pipeline.
- **JS - Resolve Role**
  - *Type & Technical Role:* `n8n-nodes-base.code` (Data Transformation)
  - *Configuration:* Extracts the customer array from Stripe output, verifies roles against valid definitions (`admin`, `user`, `readonly`), defaults missing entries to `readonly`, and flags authentication status.
  - *Key Expressions:* JavaScript array extraction and mapping.
  - *Connections:* Input: `HTTP Request - Stripe: Find Customer By API Key`; Output: `IF - Authenticated`.
- **IF - Authenticated**
  - *Type & Technical Role:* `n8n-nodes-base.if` (Flow Control)
  - *Configuration:* Evaluates whether the request is successfully authenticated.
  - *Key Expressions:* `={{ $json.authenticated === true }}`
  - *Connections:* Input: `JS - Resolve Role`; Outputs: True path to `JS - Check Route Permission`, False path to `Respond - Unauthorized (401)`.
- **Respond - Unauthorized (401)**
  - *Type & Technical Role:* `n8n-nodes-base.respondToWebhook` (Response)
  - *Configuration:* Responds immediately with HTTP status code `401`.
  - *Key Expressions:* Response Body = `={{ JSON.stringify({ success: false, error: 'invalid_api_key' }) }}`
  - *Connections:* Input: `IF - Authenticated` (False); Output: None.

#### 2.3 RBAC & Rate Limiting
This block matches the requested route against authorized user roles and increments a time-windowed counter in Redis to enforce per-role velocity limits.

- **JS - Check Route Permission**
  - *Type & Technical Role:* `n8n-nodes-base.code` (Data Transformation)
  - *Configuration:* Looks up the requested route in the route permission map to determine if the user's role is authorized.
  - *Key Expressions:* JavaScript dictionary lookups.
  - *Connections:* Input: `IF - Authenticated` (True); Output: `IF - Authorized`.
- **IF - Authorized**
  - *Type & Technical Role:* `n8n-nodes-base.if` (Flow Control)
  - *Configuration:* Splits pipeline based on whether the resolved role has permissions for the target route.
  - *Key Expressions:* `={{ $json.authorized === true }}`
  - *Connections:* Input: `JS - Check Route Permission`; Outputs: True path to `Redis - Increment Request Counter`, False path to `Respond - Forbidden (403)`.
- **Respond - Forbidden (403)**
  - *Type & Technical Role:* `n8n-nodes-base.respondToWebhook` (Response)
  - *Configuration:* Responds immediately with HTTP status code `403`.
  - *Key Expressions:* Response Body includes authorization reasons, role, and target route.
  - *Connections:* Input: `IF - Authorized` (False); Output: None.
- **Redis - Increment Request Counter**
  - *Type & Technical Role:* `n8n-nodes-base.redis` (Database / Cache)
  - *Configuration:* Performs an `incr` operation on a time-bucketed Redis key with an expiration TTL matching the rate limit window seconds.
  - *Key Expressions:* Key format: `=ratelimit:{{ $json.apiKey }}:{{ Math.floor(Date.now() / 1000 / $json.rateLimitWindowSeconds) }}`
  - *Connections:* Input: `IF - Authorized` (True); Output: `JS - Evaluate Rate Limit`.
  - *Edge Cases / Failures:* Redis unreachable. `continueOnFail` is enabled.
- **JS - Evaluate Rate Limit**
  - *Type & Technical Role:* `n8n-nodes-base.code` (Data Transformation)
  - *Configuration:* Compares Redis increment count with the maximum allowed limit for the specific user role, generating retry-after headers and remaining count metrics.
  - *Key Expressions:* JavaScript math operations.
  - *Connections:* Input: `Redis - Increment Request Counter`; Output: `IF - Within Rate Limit`.
- **IF - Within Rate Limit**
  - *Type & Technical Role:* `n8n-nodes-base.if` (Flow Control)
  - *Configuration:* Evaluates if current request volume is equal to or below the maximum permitted allocation.
  - *Key Expressions:* `={{ $json.withinRateLimit === true }}`
  - *Connections:* Input: `JS - Evaluate Rate Limit`; Outputs: True path to `JS - Resolve Target Workflow`, False path to `Respond - Rate Limited (429)`.
- **Respond - Rate Limited (429)**
  - *Type & Technical Role:* `n8n-nodes-base.respondToWebhook` (Response)
  - *Configuration:* Responds with HTTP status code `429` and sets a `Retry-After` header.
  - *Key Expressions:* Header value `Retry-After` = `={{ $json.retryAfterSeconds }}`
  - *Connections:* Input: `IF - Within Rate Limit` (False); Output: None.

#### 2.4 Internal Dispatch & Client Response
This block resolves internal network endpoints, forwards payload data packaged with security context variables, builds wrapping response structures, and responds to the origin client.

- **JS - Resolve Target Workflow**
  - *Type & Technical Role:* `n8n-nodes-base.code` (Data Transformation)
  - *Configuration:* Retrieves the matching target webhook URL from the configuration's workflow map based on the requested route.
  - *Key Expressions:* JavaScript map retrieval.
  - *Connections:* Input: `IF - Within Rate Limit` (True); Output: `HTTP Request - Call Internal Workflow`.
- **HTTP Request - Call Internal Workflow**
  - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (Integration)
  - *Configuration:* Sends a `POST` request to the internal workflow webhook URL containing the original payload wrapped alongside a `_gateway` context object.
  - *Key Expressions:* JSON body serialization appending customer ID, role, and route metadata.
  - *Connections:* Input: `JS - Resolve Target Workflow`; Output: `JS - Build Gateway Response`.
  - *Edge Cases / Failures:* Internal backend timeouts or workflow errors. `continueOnFail` is enabled.
- **JS - Build Gateway Response**
  - *Type & Technical Role:* `n8n-nodes-base.code` (Data Transformation)
  - *Configuration:* Wraps the internal system response inside a unified schema alongside rate-limit monitoring details (limit, remaining capacity, window size).
  - *Key Expressions:* JavaScript object construction.
  - *Connections:* Input: `HTTP Request - Call Internal Workflow`; Output: `Respond - Success`.
- **Respond - Success**
  - *Type & Technical Role:* `n8n-nodes-base.respondToWebhook` (Response)
  - *Configuration:* Sends success JSON response back to the API client, appending tracking metadata headers.
  - *Key Expressions:* 
    - Header `X-RateLimit-Limit`: `={{ $json.rateLimit.limit }}`
    - Header `X-RateLimit-Remaining`: `={{ $json.rateLimit.remaining }}`
  - *Connections:* Input: `JS - Build Gateway Response`; Output: None.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note - Overview | `stickyNote` | Documentation placeholder | None | None | ## n8n API Gateway - RBAC + Rate Limiting... |
| Sticky Note - Auth & Role Resolution | `stickyNote` | Documentation placeholder | None | None | ## 1. Trigger, Config & Authentication... |
| Sticky Note - RBAC Authorization | `stickyNote` | Documentation placeholder | None | None | ## 2. RBAC Authorization... |
| Sticky Note - Redis Rate Limiting | `stickyNote` | Documentation placeholder | None | None | ## 3. Rate Limiting (Redis, per-role fixed window)... |
| Sticky Note - Dispatch & Response | `stickyNote` | Documentation placeholder | None | None | ## 4. Dispatch to Internal Workflow & Response... |
| Webhook - Gateway Entry | `webhook` | API Gateway Entrypoint | None | Set - Gateway Config | ## n8n API Gateway - RBAC + Rate Limiting... <br> ## 1. Trigger, Config & Authentication |
| Set - Gateway Config | `set` | Sets routing & rate limit settings | Webhook - Gateway Entry | HTTP Request - Stripe: Find Customer By API Key | ## n8n API Gateway - RBAC + Rate Limiting... <br> ## 1. Trigger, Config & Authentication |
| HTTP Request - Stripe: Find Customer By API Key | `httpRequest` | Finds Stripe user by API key | Set - Gateway Config | JS - Resolve Role | ## n8n API Gateway - RBAC + Rate Limiting... <br> ## 1. Trigger, Config & Authentication |
| JS - Resolve Role | `code` | Resolves and validates user role | HTTP Request - Stripe: Find Customer By API Key | IF - Authenticated | ## n8n API Gateway - RBAC + Rate Limiting... <br> ## 1. Trigger, Config & Authentication |
| IF - Authenticated | `if` | Validates API key authenticity | JS - Resolve Role | JS - Check Route Permission, Respond - Unauthorized (401) | ## n8n API Gateway - RBAC + Rate Limiting... <br> ## 1. Trigger, Config & Authentication |
| JS - Check Route Permission | `code` | Evaluates RBAC permissions | IF - Authenticated | IF - Authorized | ## n8n API Gateway - RBAC + Rate Limiting... <br> ## 2. RBAC Authorization |
| IF - Authorized | `if` | Routes based on route permissions | JS - Check Route Permission | Redis - Increment Request Counter, Respond - Forbidden (403) | ## n8n API Gateway - RBAC + Rate Limiting... <br> ## 2. RBAC Authorization |
| Redis - Increment Request Counter | `redis` | Increments Redis velocity counter | IF - Authorized | JS - Evaluate Rate Limit | ## n8n API Gateway - RBAC + Rate Limiting... <br> ## 3. Rate Limiting (Redis, per-role fixed window) |
| JS - Evaluate Rate Limit | `code` | Compares counter against limits | Redis - Increment Request Counter | IF - Within Rate Limit | ## n8n API Gateway - RBAC + Rate Limiting... <br> ## 3. Rate Limiting (Redis, per-role fixed window) |
| IF - Within Rate Limit | `if` | Splits on rate limit thresholds | JS - Evaluate Rate Limit | JS - Resolve Target Workflow, Respond - Rate Limited (429) | ## n8n API Gateway - RBAC + Rate Limiting... <br> ## 3. Rate Limiting (Redis, per-role fixed window) |
| JS - Resolve Target Workflow | `code` | Resolves target webhook endpoint | IF - Within Rate Limit | HTTP Request - Call Internal Workflow | ## n8n API Gateway - RBAC + Rate Limiting... <br> ## 4. Dispatch to Internal Workflow & Response |
| HTTP Request - Call Internal Workflow | `httpRequest` | Dispatches to backend workflow | JS - Resolve Target Workflow | JS - Build Gateway Response | ## n8n API Gateway - RBAC + Rate Limiting... <br> ## 4. Dispatch to Internal Workflow & Response |
| JS - Build Gateway Response | `code` | Wraps response with headers/meta | HTTP Request - Call Internal Workflow | Respond - Success | ## n8n API Gateway - RBAC + Rate Limiting... <br> ## 4. Dispatch to Internal Workflow & Response |
| Respond - Success | `respondToWebhook` | Returns success payload to client | JS - Build Gateway Response | None | ## n8n API Gateway - RBAC + Rate Limiting... <br> ## 4. Dispatch to Internal Workflow & Response |
| Respond - Unauthorized (401) | `respondToWebhook` | Returns 401 error response | IF - Authenticated | None | ## n8n API Gateway - RBAC + Rate Limiting... <br> ## 1. Trigger, Config & Authentication |
| Respond - Forbidden (403) | `respondToWebhook` | Returns 403 error response | IF - Authorized | None | ## n8n API Gateway - RBAC + Rate Limiting... <br> ## 2. RBAC Authorization |
| Respond - Rate Limited (429) | `respondToWebhook` | Returns 429 error response | IF - Within Rate Limit | None | ## n8n API Gateway - RBAC + Rate Limiting... <br> ## 3. Rate Limiting (Redis, per-role fixed window) |

---

### 4. Reproducing the Workflow from Scratch

Follow these manual steps to rebuild the workflow inside an n8n canvas:

1. **Create Webhook Node (`Webhook - Gateway Entry`)**
   - Type: `n8n-nodes-base.webhook`
   - Parameters: Set HTTP Method to `POST`, path to `gateway`, response mode to `responseNode`.
2. **Create Configuration Node (`Set - Gateway Config`)**
   - Type: `n8n-nodes-base.set`
   - Parameters: Add custom assignment fields for `apiKey`, `targetRoute`, `method`, `payload`, `rateLimitWindowSeconds` (number: `60`), `rateLimits` (object defining limits for `admin`, `user`, `readonly`), `routePermissions` (object mapping routes to arrays of allowed roles), and `internalWorkflowMap` (object mapping routes to target URLs). Connect input from Step 1.
3. **Create Stripe Customer Lookup Node (`HTTP Request - Stripe: Find Customer By API Key`)**
   - Type: `n8n-nodes-base.httpRequest`
   - Credentials: Configure Stripe API credentials.
   - Parameters: Method `GET`, URL `https://api.stripe.com/v1/customers/search`, query parameter `query` set to lookup metadata API key values. Enable `Continue On Fail`. Connect input from Step 2.
4. **Create Role Resolution Node (`JS - Resolve Role`)**
   - Type: `n8n-nodes-base.code`
   - Parameters: Paste custom JavaScript code to extract customer metadata, normalize roles, and assign default values (`readonly`). Connect input from Step 3.
5. **Create Authentication Branching Node (`IF - Authenticated`)**
   - Type: `n8n-nodes-base.if`
   - Parameters: Set condition to evaluate `{{ $json.authenticated === true }}`. Connect input from Step 4.
6. **Create Unauthorized Response Node (`Respond - Unauthorized (401)`)**
   - Type: `n8n-nodes-base.respondToWebhook`
   - Parameters: Set response code to `401`, response body to JSON error string (`invalid_api_key`). Connect input to the *False* output of Step 5.
7. **Create Permission Check Node (`JS - Check Route Permission`)**
   - Type: `n8n-nodes-base.code`
   - Parameters: Add script validating requested route validity and role clearance. Connect input to the *True* output of Step 5.
8. **Create Authorization Branching Node (`IF - Authorized`)**
   - Type: `n8n-nodes-base.if`
   - Parameters: Set condition to evaluate `{{ $json.authorized === true }}`. Connect input from Step 7.
9. **Create Forbidden Response Node (`Respond - Forbidden (403)`)**
   - Type: `n8n-nodes-base.respondToWebhook`
   - Parameters: Set response code to `403`, response body returning authorization metadata. Connect input to the *False* output of Step 8.
10. **Create Redis Increment Node (`Redis - Increment Request Counter`)**
    - Type: `n8n-nodes-base.redis`
    - Credentials: Configure Redis connection parameters.
    - Parameters: Operation `incr`, set dynamic key pattern matching API keys and window increments, set TTL expiration matching rate limit window seconds. Enable `Continue On Fail`. Connect input to the *True* output of Step 8.
11. **Create Rate Limit Evaluation Node (`JS - Evaluate Rate Limit`)**
    - Type: `n8n-nodes-base.code`
    - Parameters: Add script checking usage counts against role configurations. Connect input from Step 10.
12. **Create Rate Limit Branching Node (`IF - Within Rate Limit`)**
    - Type: `n8n-nodes-base.if`
    - Parameters: Set condition to evaluate `{{ $json.withinRateLimit === true }}`. Connect input from Step 11.
13. **Create Rate Limited Response Node (`Respond - Rate Limited (429)`)**
    - Type: `n8n-nodes-base.respondToWebhook`
    - Parameters: Set response code to `429`, configure `Retry-After` header. Connect input to the *False* output of Step 12.
14. **Create Target Workflow Resolver Node (`JS - Resolve Target Workflow`)**
    - Type: `n8n-nodes-base.code`
    - Parameters: Add script mapping route values to internal URLs. Connect input to the *True* output of Step 12.
15. **Create Internal Dispatch Node (`HTTP Request - Call Internal Workflow`)**
    - Type: `n8n-nodes-base.httpRequest`
    - Parameters: Method `POST`, URL mapped from target URL data, JSON body payload appending context objects (`_gateway`). Enable `Continue On Fail`. Connect input from Step 14.
16. **Create Gateway Response Builder Node (`JS - Build Gateway Response`)**
    - Type: `n8n-nodes-base.code`
    - Parameters: Add script wrapping response items and tracking stats. Connect input from Step 15.
17. **Create Success Response Node (`Respond - Success`)**
    - Type: `n8n-nodes-base.respondToWebhook`
    - Parameters: Set response mode to JSON, configure headers (`X-RateLimit-Limit`, `X-RateLimit-Remaining`). Connect input from Step 16.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Stripe Metadata Requirements | Ensure all Stripe customer objects include `metadata.apiKey` and `metadata.role` (admin, user, or readonly). |
| Redis Instance Requirements | Ensure the Redis instance supports atomic `INCR` operations and is accessible from the n8n runner environment. |
| Sub-Workflow Context Integration | Internal target workflows receive an appended `_gateway` object containing `customerId`, `role`, and `route` parameters for downstream telemetry. |
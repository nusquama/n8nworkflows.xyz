Send Telegram Notification for New WooCommerce Orders

https://n8nworkflows.xyz/workflows/send-telegram-notification-for-new-woocommerce-orders-4818


# Send Telegram Notification for New WooCommerce Orders

### 1. Workflow Overview

This workflow automates the notification process for new orders placed on a WooCommerce store by sending detailed order information directly to a Telegram chat or group. It is designed for store administrators or support teams who want real-time order alerts via Telegram.

The workflow consists of two main logical blocks:

- **1.1 WooCommerce Trigger:** Listens for new order creation events from WooCommerce.
- **1.2 Telegram Notification:** Sends a richly formatted message about the new order to a specified Telegram chat, including an interactive button for quick order access.

A **Sticky Note** provides setup instructions and clarifies configuration details.

---

### 2. Block-by-Block Analysis

#### 1.1 WooCommerce Trigger

- **Overview:**  
  This block listens for the creation of new orders on a WooCommerce store via a webhook. Once a new order is detected, it triggers the workflow with order data.

- **Nodes Involved:**  
  - WooCommerce Trigger

- **Node Details:**  

  - **Node Name:** WooCommerce Trigger  
  - **Type:** `wooCommerceTrigger` (Trigger node)  
  - **Technical Role:** Event listener for WooCommerce order creation events.  
  - **Configuration Choices:**  
    - Event: `order.created` (fires when a new order is placed).  
    - Credentials: Uses WooCommerce API credentials containing Consumer Key, Secret, and Base URL.  
  - **Key Expressions/Variables:** None (trigger node directly outputs webhook data).  
  - **Input Connections:** None (trigger node initiates the workflow).  
  - **Output Connections:** Connected to the Telegram node.  
  - **Version-Specific Requirements:** Version 1; ensure WooCommerce API supports webhooks for order creation.  
  - **Potential Failures:**  
    - Authentication errors due to invalid API credentials.  
    - Network connectivity or webhook registration issues.  
    - WooCommerce REST API disabled or restricted.  
  - **Sub-Workflow:** None.

#### 1.2 Telegram Notification

- **Overview:**  
  This block sends a formatted message to a Telegram chat or group containing the new order's details, including an order summary and a clickable button linking to the order page.

- **Nodes Involved:**  
  - Telegram

- **Node Details:**  

  - **Node Name:** Telegram  
  - **Type:** `telegram` (Action node)  
  - **Technical Role:** Sends messages and interactive elements to a Telegram chat using Telegram Bot API.  
  - **Configuration Choices:**  
    - `chatId`: Set to a specific Telegram user or group ID (e.g., "8025476048").  
    - `text`: Composed dynamically using n8n expressions pulling from the WooCommerce order JSON payload:  
      - Order ID (`{{$json.id}}`)  
      - Total amount (`{{$json.total}}`) in USD  
      - Order creation date (`{{$json.date_created}}`)  
      - Order status (`{{$json.status}}`)  
      - Product list: Iterates over `line_items` to list each product name, quantity, and total price.  
    - `replyMarkup`: Configured as an inline keyboard with a button labeled "🔗 View Order" that opens the order payment URL (`{{$json.payment_url}}`).  
    - Credentials: Uses Telegram API credentials linked to a Telegram bot.  
  - **Key Expressions/Variables:**  
    - Inline JavaScript expression for products list formatting:  
      ```js
      $json.line_items.map(item => `- ${item.name} × ${item.quantity} = ${item.total} USD`).join('\n')
      ```  
    - Dynamic URL for button using `{{$json.payment_url}}`.  
  - **Input Connections:** Connected from WooCommerce Trigger node.  
  - **Output Connections:** None (terminal node).  
  - **Version-Specific Requirements:** Version 1.2; supports inline keyboard feature introduced in Telegram API v4.0+ (ensure bot API compatibility).  
  - **Potential Failures:**  
    - Incorrect or unauthorized `chatId` causing message delivery failure.  
    - Telegram Bot API rate limits or downtime.  
    - Expression evaluation errors if expected JSON fields are missing or malformed.  
    - `payment_url` missing or invalid URL.  
  - **Sub-Workflow:** None.

#### Sticky Note: Setup Instructions

- **Overview:**  
  Provides detailed instructions for configuring and customizing the workflow.

- **Nodes Involved:**  
  - Sticky Note

- **Node Details:**  

  - **Node Name:** Sticky Note  
  - **Type:** `stickyNote` (Informational node)  
  - **Technical Role:** Documentation and guidance for users.  
  - **Content Highlights:**  
    - Replace `chatId` in the Telegram node with your Telegram user or group ID.  
    - Add the Telegram bot to the relevant group if used.  
    - Update WooCommerce credentials with valid API keys and base URL.  
    - Ensure WooCommerce REST API access is enabled.  
    - Currency shown as USD by default; modifiable if needed.  
    - Workflow triggers on new WooCommerce orders and sends a message with order details and a view order button.  
  - **Input/Output Connections:** None.  
  - **Version-Specific Requirements:** None.  
  - **Potential Failures:** None (informational only).  
  - **Sub-Workflow:** None.

---

### 3. Summary Table

| Node Name           | Node Type              | Functional Role               | Input Node(s)       | Output Node(s)     | Sticky Note                                                                                      |
|---------------------|------------------------|------------------------------|---------------------|--------------------|-------------------------------------------------------------------------------------------------|
| WooCommerce Trigger  | wooCommerceTrigger     | Listens for new WooCommerce orders | None                | Telegram           | Replace WooCommerce API credentials with Consumer Key, Secret, Base URL. Ensure REST API access.|
| Telegram            | telegram               | Sends order notification to Telegram | WooCommerce Trigger | None               | Replace `chatId` with your Telegram user/group ID. Add bot to group. Uses dynamic order data.   |
| Sticky Note         | stickyNote             | Provides setup and usage instructions | None                | None               | Contains detailed setup steps, currency notes, and workflow purpose description.                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create WooCommerce Trigger Node:**  
   - Add a new node of type `WooCommerce Trigger`.  
   - Set the event to `order.created`.  
   - Configure WooCommerce API credentials: provide Consumer Key, Consumer Secret, and Base URL of your WooCommerce store.  
   - Ensure WooCommerce REST API and webhook permissions are enabled on your store.

2. **Create Telegram Node:**  
   - Add a new node of type `Telegram`.  
   - Enter your Telegram Bot API credentials (Bot Token).  
   - Set the `chatId` parameter to your personal Telegram user ID or group ID where notifications should be sent.  
   - Configure the message `text` field using expressions to include:  
     ```
     🛒 New Order Received!

     🆔 Order #: {{$json.id}}  
     💰 Total: {{$json.total}} USD  
     📅 Date: {{$json.date_created}}  
     🔖 Status: {{$json.status}}  

     🛍️ Products:
     {{ $json.line_items.map(item => `- ${item.name} × ${item.quantity} = ${item.total} USD`).join('\n') }}

     👁️‍🗨️ Click the button below to view this order.
     ```
   - Enable `replyMarkup` as `inlineKeyboard`.  
   - Add a button with text "🔗 View Order" that opens the URL from `{{$json.payment_url}}` using Telegram's `web_app` or URL button feature.

3. **Connect Nodes:**  
   - Connect the output of the WooCommerce Trigger node to the input of the Telegram node.

4. **Validate Credentials:**  
   - Test WooCommerce API credentials by ensuring the webhook is correctly registered and triggers on new orders.  
   - Test Telegram Bot credentials by sending a test message or manually triggering the Telegram node with a sample payload.

5. **Save and Activate Workflow:**  
   - Save the workflow and activate it to start listening to new WooCommerce orders and sending Telegram notifications automatically.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                      | Context or Link                                                                 |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------|
| The order notification message uses dynamic expressions to format order details and the product list, ensuring clear and readable notifications.                               | Workflow message formatting detail                                             |
| The inline keyboard button opens the order payment URL dynamically, eliminating the need to hardcode domain URLs and improving maintainability.                                | Telegram inline button setup                                                   |
| Ensure your Telegram bot has permission to send messages to the target chat or group; if sending to a group, the bot must be added to the group with appropriate permissions.    | Telegram bot permission requirement                                            |
| WooCommerce REST API must be enabled and accessible externally for the webhook to function correctly.                                                                            | WooCommerce API access requirement                                            |
| Currency is set to USD for international compatibility, but it can be changed in the Telegram node's expressions if necessary.                                                 | Currency configuration note                                                   |
| For more details on WooCommerce API and webhook setup, refer to WooCommerce official documentation: https://woocommerce.github.io/woocommerce-rest-api-docs/#webhooks             | WooCommerce API documentation link                                             |
| Learn about Telegram bots and inline keyboards here: https://core.telegram.org/bots/api#inlinekeyboardmarkup                                                                    | Telegram Bot API documentation link                                           |
| For managing credentials securely in n8n, consult: https://docs.n8n.io/credentials/                                                                                              | n8n credentials management documentation                                       |

---

**Disclaimer:** The text above derives exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected material. All handled data is legal and public.
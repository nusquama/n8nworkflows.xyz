Automate Service Booking & Payment with WhatsApp and Xendit

https://n8nworkflows.xyz/workflows/automate-service-booking---payment-with-whatsapp-and-xendit-6474


# Automate Service Booking & Payment with WhatsApp and Xendit

### 1. Workflow Overview

This workflow automates the process of service booking and payment for a makeup service business using WhatsApp for communication and Xendit for payment invoicing. It receives booking requests via a webhook, processes and calculates pricing based on booking details, generates payment invoices through Xendit, and communicates booking confirmations and payment links back to customers on WhatsApp.

**Target Use Cases:**  
- Automating makeup service bookings submitted via an external form or system.  
- Calculating dynamic pricing with add-ons, transport fees, and partial payments (down payments).  
- Generating payment invoices with expiry and redirect URLs via Xendit API.  
- Sending personalized WhatsApp messages with booking details and payment links using GoWhatsApp API.  

**Logical Blocks:**  
- **1.1 Input Reception**: Receives booking data via webhook.  
- **1.2 Booking Data Processing**: Extracts and formats booking details, constructs booking ID.  
- **1.3 Price Calculation**: Calculates total price, add-ons, transport fees, down payment logic, and formats prices accordingly.  
- **1.4 Invoice Generation**: Sends booking and payment data to Xendit API to generate an invoice.  
- **1.5 WhatsApp Communication**: Manages typing indicators and sends booking and payment confirmation messages to customers on WhatsApp.  

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
Receives incoming booking requests through an HTTP POST webhook endpoint. This node is the entry point of the workflow.

**Nodes Involved:**  
- `Webhook`

**Node Details:**  
- **Type:** Webhook (n8n built-in)  
- **Configuration:** HTTP POST method, path `/makeup-booking`. Response mode set to respond immediately after processing.  
- **Expressions/Variables:** None directly; raw request JSON is passed on.  
- **Connections:** Output to `Process Booking` node.  
- **Edge Cases:** Invalid or incomplete POST payloads; webhook availability and security considerations (e.g., authentication not configured here).  
- **Version Requirements:** n8n 2.x or higher recommended for current webhook features.

---

#### 1.2 Booking Data Processing

**Overview:**  
Parses incoming JSON body and extracts key booking fields into structured parameters, including a unique booking ID derived from customer name, event date, and makeup type.

**Nodes Involved:**  
- `Process Booking`

**Node Details:**  
- **Type:** Set node (n8n built-in)  
- **Configuration:** Assigns variables from incoming JSON body fields such as `nama`, `noHp`, `tanggal`, `jam`, `tempatAcara`, `jenisMakeup`, `addOn`, `tipePembayaran`. Constructs `bookingId` as a string combining "Khaisa_BOOK_" prefix plus customer name, date, and makeup type.  
- **Expressions:** Uses expressions like `={{ $json.body.nama }}`, templated string construction for `bookingId`.  
- **Connections:** Output to `Calculate Price`.  
- **Edge Cases:** Missing or malformed input fields may cause empty or invalid booking IDs; no validation logic present here.  
- **Version Requirements:** n8n 2.x for advanced expression support.

---

#### 1.3 Price Calculation

**Overview:**  
Executes JavaScript code to calculate pricing components based on makeup type, service type (e.g., homeservice transport fee), add-ons, and payment type (full or down payment). Also formats phone numbers and currency strings for further processing.

**Nodes Involved:**  
- `Calculate Price`

**Node Details:**  
- **Type:** Code node (JavaScript)  
- **Configuration:**  
  - Defines base prices for various makeup types.  
  - Formats WhatsApp number from local format (leading 0) to international (prefix 62) and removes non-numeric characters.  
  - Adds transport fee for homeservice.  
  - Calculates add-on fee at 10% of base price if add-ons present.  
  - Sets invoice amount to a down payment (50,000 IDR) if payment type includes "DP" or "Booking".  
  - Formats prices for display (Indonesian Rupiah) and for Xendit (numeric string).  
  - Creates unique external invoice ID by stripping "BOOK_" prefix.  
  - Generates invoice description text with DP indicator if applicable.  
  - Sets invoice expiry time to 24 hours ahead.  
  - Calculates remaining payment if down payment applied.  
- **Expressions:** Embedded JS code iterating over all input items.  
- **Connections:** Output to `Generate Invoice`.  
- **Edge Cases:** Unknown makeup types default to 200,000 IDR; malformed phone numbers may result in incorrect formatting; missing payment types could affect DP logic; time zone issues possible with expiry timestamp.  
- **Version Requirements:** n8n 2.x for advanced code node features.

---

#### 1.4 Invoice Generation

**Overview:**  
Sends a POST request to Xendit's Invoice API to create a payment invoice using the calculated price and booking details, including customer info and invoice metadata.

**Nodes Involved:**  
- `Generate Invoice`

**Node Details:**  
- **Type:** HTTP Request node (n8n built-in)  
- **Configuration:**  
  - URL: `https://api.xendit.co/v2/invoices`  
  - Method: POST  
  - Authentication: HTTP Basic Auth with Xendit API key as username, password empty.  
  - JSON body dynamically populated with invoice details (`external_id`, `amount`, `description`, `invoice_duration`, customer name, phone, static email, redirect URLs for success/failure, currency IDR, item details, metadata).  
- **Expressions:** Template syntax to inject JSON fields from previous node outputs.  
- **Connections:** Outputs to `Respond to Webhook` and `Start Typing` nodes.  
- **Edge Cases:** API errors (authentication failure, invalid parameters, network issues), static email placeholder used (may cause invoice email notifications to fail or be misrouted).  
- **Version Requirements:** n8n 2.x or higher for HTTP node authentication features.

---

#### 1.5 WhatsApp Communication

**Overview:**  
Manages WhatsApp chat presence (typing indicators) and sends the final booking confirmation message with payment link to the customer’s WhatsApp number.

**Nodes Involved:**  
- `Start Typing`  
- `Delay Typing`  
- `Stop Typing`  
- `Send Message`

**Node Details:**  
- **Type:** GoWhatsApp API Community Node (`@aldinokemal2104/n8n-nodes-gowa.gowa`)  
- **Configuration:**  
  - `Start Typing`: Sends typing presence start to formatted WhatsApp number.  
  - `Delay Typing`: Waits a random 3–8 seconds to simulate typing delay.  
  - `Stop Typing`: Sends typing presence stop signal.  
  - `Send Message`: Sends a templated message including booking details and payment link (`invoice_url`) from Xendit.  
- **Expressions:** Phone number formatted in earlier steps; message templates use multiple previous node outputs for dynamic content.  
- **Credentials:** Uses stored GoWhatsApp API credentials connected to a WhatsApp Web multi-device container.  
- **Connections:** Sequential flow from `Generate Invoice` → `Start Typing` → `Delay Typing` → `Stop Typing` → `Send Message`.  
- **Edge Cases:** WhatsApp API connectivity issues, invalid phone number format, message sending failures, API rate limits.  
- **Version Requirements:** Requires community node installation and configured GoWhatsApp API server.

---

#### Sticky Notes (Contextual Reference)

- Sticky Note (positioned left): Shows a screenshot of the booking form for reference.  
- Sticky Note1: Instructions for obtaining and setting up Xendit API key with HTTP Basic Auth.  
- Sticky Note2: Instructions for setting up GoWhatsApp API server and node configuration in n8n.

---

### 3. Summary Table

| Node Name        | Node Type                          | Functional Role                   | Input Node(s)      | Output Node(s)                  | Sticky Note                                                                                                           |
|------------------|----------------------------------|---------------------------------|--------------------|-------------------------------|-----------------------------------------------------------------------------------------------------------------------|
| Webhook          | Webhook                          | Receive booking request          | -                  | Process Booking                |                                                                                                                       |
| Process Booking   | Set                              | Extract and structure booking data | Webhook            | Calculate Price                |                                                                                                                       |
| Calculate Price   | Code                             | Calculate pricing and format data | Process Booking     | Generate Invoice               |                                                                                                                       |
| Generate Invoice  | HTTP Request                     | Create payment invoice via Xendit | Calculate Price     | Respond to Webhook, Start Typing | Sticky Note1: Xendit API key setup and usage instructions                                                               |
| Respond to Webhook| Respond to Webhook               | Send response to original HTTP request | Generate Invoice  | -                             |                                                                                                                       |
| Start Typing     | GoWhatsApp (community node)       | Send typing indicator start     | Generate Invoice    | Delay Typing                  | Sticky Note2: Setup instructions for GoWhatsApp API server and node configuration                                      |
| Delay Typing     | Wait                             | Simulate typing delay            | Start Typing        | Stop Typing                   | Sticky Note2 (shared with other GoWhatsApp nodes)                                                                      |
| Stop Typing      | GoWhatsApp (community node)       | Send typing indicator stop      | Delay Typing        | Send Message                  | Sticky Note2 (shared)                                                                                                  |
| Send Message     | GoWhatsApp (community node)       | Send booking confirmation message | Stop Typing         | -                             | Sticky Note2 (shared)                                                                                                  |
| Sticky Note      | Sticky Note                      | Booking form screenshot          | -                  | -                             | ![form](https://raw.githubusercontent.com/khmuhtadin/n8n-template/main/Additional%20FE/Automate%20Booking%20with%20Xendit%20/Images/Screenshot%202025-07-25%20at%2021.09.06.png) |
| Sticky Note1     | Sticky Note                      | Xendit API key setup instructions | -                  | -                             | Xendit API key setup steps with useful links                                                                           |
| Sticky Note2     | Sticky Note                      | GoWhatsApp API setup instructions | -                  | -                             | GoWhatsApp API setup steps with useful links                                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `makeup-booking`  
   - Response Mode: `responseNode`  
   - Purpose: Receive booking data via HTTP POST.

2. **Create Set Node - "Process Booking"**  
   - Type: Set  
   - Assign fields from webhook JSON body:  
     - `bookingId`: `"Khaisa_BOOK_{{ $json.body.nama }}-{{ $json.body.tanggal }}-{{ $json.body.jenisMakeup }}"`  
     - `customerName`: `{{ $json.body.nama }}`  
     - `whatsappNumber`: `{{ $json.body.noHp }}`  
     - `eventDate`: `{{ $json.body.tanggal }}`  
     - `eventTime`: `{{ $json.body.jam }}`  
     - `location`: `{{ $json.body.tempatAcara }}`  
     - `makeupType`: `{{ $json.body.jenisMakeup }}`  
     - `addOns`: `{{ $json.body.addOn }}`  
     - `paymentType`: `{{ $json.body.tipePembayaran }}`  
   - Connect Webhook → Process Booking.

3. **Create Code Node - "Calculate Price"**  
   - Type: Code (JavaScript)  
   - Paste the provided JS logic for pricing calculation, WhatsApp number formatting, invoice description, and expiry date calculation.  
   - Connect Process Booking → Calculate Price.

4. **Create HTTP Request Node - "Generate Invoice"**  
   - Type: HTTP Request  
   - URL: `https://api.xendit.co/v2/invoices`  
   - Method: POST  
   - Authentication: HTTP Basic Auth  
     - Username: Xendit API key (obtain from [Xendit dashboard](https://dashboard.xendit.co/))  
     - Password: leave empty  
   - Body Content Type: JSON  
   - Body: Use dynamic expressions to fill invoice details as per node configuration (external_id, amount, description, customer info, redirect URLs, currency, items, metadata).  
   - Connect Calculate Price → Generate Invoice.

5. **Create Respond to Webhook Node**  
   - Type: Respond to Webhook  
   - Respond With: All Incoming Items  
   - Connect Generate Invoice → Respond to Webhook.

6. **Setup GoWhatsApp API Node Credentials**  
   - Install community node `@aldinokemal2104/n8n-nodes-gowa.gowa` via n8n community nodes.  
   - Setup GoWhatsApp API server using Docker container as per instructions:  
     ```bash
     docker run -d -p 3000:3000 aldinokemal2104/go-whatsapp-web-multidevice
     ```  
   - Access `http://localhost:3000` and scan QR code with WhatsApp.

7. **Create GoWhatsApp Node - "Start Typing"**  
   - Operation: `sendChatPresence`  
   - Phone Number: `={{ $('Calculate Price').item.json.formattedWhatsapp }}`  
   - Action: start typing  
   - Connect Generate Invoice → Start Typing.

8. **Create Wait Node - "Delay Typing"**  
   - Wait Time: Expression `={{ Math.random()*5+3 }}` seconds (random delay 3-8s)  
   - Connect Start Typing → Delay Typing.

9. **Create GoWhatsApp Node - "Stop Typing"**  
   - Operation: `sendChatPresence`  
   - Phone Number: `={{ $('Calculate Price').item.json.formattedWhatsapp }}`  
   - Action: stop typing  
   - Connect Delay Typing → Stop Typing.

10. **Create GoWhatsApp Node - "Send Message"**  
    - Operation: Send Message  
    - Phone Number: `={{ $('Calculate Price').item.json.formattedWhatsapp }}`  
    - Message: Use template with booking details and payment link, e.g.:  
      ```
      Hai, {{ $('Calculate Price').item.json.customerName }}

      Berikut Detail Booking Kamu:
      Booking Id: {{ $('Process Booking').item.json.bookingId }}
      Tipe Makeup: {{ $('Process Booking').item.json.makeupType }}
      Tanggal: {{ $('Process Booking').item.json.eventDate }} / {{ $('Process Booking').item.json.eventTime }}
      Lokasi: {{ $('Process Booking').item.json.location }}
      Tipe Pembayaran: {{ $('Process Booking').item.json.paymentType }}

      Jika Belum melakukan pembayaran di halaman form, kamu bisa melakukan pembayaran melalui link berikut ya
      {{ $('Generate Invoice').item.json.invoice_url }}
      ```
    - Connect Stop Typing → Send Message.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                      | Context or Link                                                                                                                        |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------|
| Screenshot of the booking form used to collect customer data and trigger the webhook.                                                                                                                                                                                                                            | ![Booking Form Screenshot](https://raw.githubusercontent.com/khmuhtadin/n8n-template/main/Additional%20FE/Automate%20Booking%20with%20Xendit%20/Images/Screenshot%202025-07-25%20at%2021.09.06.png) |
| How to get Xendit API key: Register/Login at Xendit Dashboard, generate API key under Settings → API Keys, use Basic Auth in HTTP Request node with the API key as username and empty password. Documentation links: [Xendit API](https://developers.xendit.co/), [Invoice API](https://developers.xendit.co/api-reference/#create-invoice) | Xendit Dashboard: https://dashboard.xendit.co/                                                                                         |
| GoWhatsApp API setup instructions: Install GoWhatsApp docker container, scan QR code for WhatsApp multi-device session, use community node in n8n with API URL `http://localhost:3000`. Docs: [GoWhatsApp GitHub](https://github.com/aldinokemal/go-whatsapp-web-multidevice), [API Docs](https://github.com/aldinokemal/go-whatsapp-web-multidevice#api-docs) | GoWhatsApp Docker: `docker run -d -p 3000:3000 aldinokemal2104/go-whatsapp-web-multidevice`                                            |

---

**Disclaimer:** The provided text derives exclusively from an automated workflow created with n8n, respecting all relevant content policies and containing no illegal or offensive elements. All processed data are legal and public.
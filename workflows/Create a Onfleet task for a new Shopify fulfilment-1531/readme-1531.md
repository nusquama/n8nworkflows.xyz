Create a Onfleet task for a new Shopify fulfilment

https://n8nworkflows.xyz/workflows/create-a-onfleet-task-for-a-new-shopify-fulfilment-1531


# Create a Onfleet task for a new Shopify fulfilment

### 1. Workflow Overview

This workflow automates the creation of an Onfleet delivery task whenever a new fulfillment event occurs in a Shopify store. It is designed for e-commerce businesses using Shopify and Onfleet to streamline last-mile delivery operations by automatically initiating delivery tasks without manual intervention.

**Logical Blocks:**

- **1.1 Shopify Fulfillment Event Reception:**  
  Listens for new fulfillment creation events on a Shopify store.

- **1.2 Onfleet Task Creation:**  
  Uses the fulfillment data to create a delivery task in Onfleet, enabling automated dispatch and route planning.

---

### 2. Block-by-Block Analysis

#### 1.1 Shopify Fulfillment Event Reception

- **Overview:**  
  This block captures fulfillment creation events from Shopify in real-time, triggering the workflow when an order fulfillment is created.

- **Nodes Involved:**  
  - Shopify Trigger

- **Node Details:**

  - **Shopify Trigger**  
    - **Type:** Trigger node for Shopify events  
    - **Technical Role:** Listens to Shopify webhook events for fulfillment creation  
    - **Configuration:**  
      - Topic set to `fulfillments/create` to listen for new fulfillment events  
      - Requires Shopify API credentials configured with appropriate permissions to receive webhook data  
    - **Key Expressions/Variables:**  
      - Automatically receives the fulfillment object payload from Shopify, including order details, shipping address, and fulfillment metadata  
    - **Input/Output Connections:**  
      - No input (trigger node)  
      - Output connected to the Onfleet node  
    - **Version Requirements:**  
      - Compatible with n8n version supporting Shopify Trigger node v1  
    - **Potential Failures:**  
      - Webhook registration failure if Shopify credentials lack webhook permissions  
      - Network issues causing missed events  
      - Shopify API rate limiting or downtime  
    - **Sub-Workflow Reference:** None

#### 1.2 Onfleet Task Creation

- **Overview:**  
  This block creates a new delivery task in Onfleet using data received from the Shopify fulfillment event.

- **Nodes Involved:**  
  - Onfleet

- **Node Details:**

  - **Onfleet**  
    - **Type:** API integration node for Onfleet  
    - **Technical Role:** Creates new delivery tasks in Onfleet based on input data  
    - **Configuration:**  
      - Operation set to `create` to add a new task  
      - Additional fields left empty by default but can be customized to map more Shopify fulfillment data (e.g., recipient details, pickup/delivery times)  
      - Requires Onfleet API key credentials with permissions to create tasks  
    - **Key Expressions/Variables:**  
      - Receives fulfillment data from the Shopify Trigger node, typically including recipient address, contact info, and order details to populate the delivery task  
    - **Input/Output Connections:**  
      - Input from Shopify Trigger node  
      - No further output nodes  
    - **Version Requirements:**  
      - Compatible with n8n version supporting Onfleet node v1  
    - **Potential Failures:**  
      - Authentication errors if API key is invalid or lacks permissions  
      - Data mapping issues if required fields are missing or malformed in the fulfillment data  
      - API rate limiting or downtime  
    - **Sub-Workflow Reference:** None

---

### 3. Summary Table

| Node Name       | Node Type               | Functional Role                      | Input Node(s)    | Output Node(s) | Sticky Note                                                                                   |
|-----------------|-------------------------|------------------------------------|------------------|----------------|----------------------------------------------------------------------------------------------|
| Shopify Trigger | Shopify Trigger          | Listen for new Shopify fulfillments | None             | Onfleet        | Update with your own Shopify credentials                                                    |
| Onfleet         | Onfleet API Integration  | Create Onfleet delivery task         | Shopify Trigger  | None           | Update with your own Onfleet credentials. Register for an API key at https://onfleet.com/signup |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add the Shopify Trigger node:**  
   - Set node name to `Shopify Trigger`.  
   - Set “Topic” parameter to `fulfillments/create`.  
   - Configure Shopify API credentials with your Shopify store account that has permissions to access webhook events and order fulfillments.  
   - Position the node appropriately (e.g., x=240, y=440 for visual consistency).  
   - This node will act as the workflow trigger, firing when a new fulfillment is created.

3. **Add the Onfleet node:**  
   - Set node name to `Onfleet`.  
   - Choose operation `create` to create a new delivery task.  
   - Leave additional fields empty initially or map relevant Shopify fulfillment data to fields such as recipient name, phone, address, etc.  
   - Configure Onfleet API credentials using your Onfleet API key (obtain one by signing up at https://onfleet.com/signup).  
   - Position the node next to the Shopify Trigger node (e.g., x=460, y=440).

4. **Connect the nodes:**  
   - Connect the output of the `Shopify Trigger` node to the input of the `Onfleet` node.

5. **Save and activate the workflow:**  
   - Activate the workflow to start listening to Shopify fulfillment events and creating Onfleet tasks automatically.

6. **Optional customization:**  
   - Modify the Onfleet node’s additional fields to map more data from the Shopify fulfillment payload, such as scheduled delivery windows, notes, or task priority.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                          |
|----------------------------------------------------------------------------------------------|----------------------------------------|
| Use your own Shopify credentials to ensure proper webhook registration and data permissions. | Shopify Trigger node setup              |
| Register and obtain an Onfleet API key at https://onfleet.com/signup to enable task creation. | Onfleet node credential setup           |
| You can customize task creation by mapping additional Shopify fulfillment fields in the Onfleet node. | Enhancing delivery task details         |
| Onfleet provides end-to-end route planning, dispatch, communication, and analytics.          | https://onfleet.com                     |
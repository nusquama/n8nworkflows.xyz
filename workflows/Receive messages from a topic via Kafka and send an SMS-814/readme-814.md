Receive messages from a topic via Kafka and send an SMS

https://n8nworkflows.xyz/workflows/receive-messages-from-a-topic-via-kafka-and-send-an-sms-814


# Receive messages from a topic via Kafka and send an SMS

### 1. Workflow Overview

This workflow listens for incoming messages on a Kafka topic, evaluates the temperature value inside each message, and conditionally sends an SMS alert if the temperature exceeds a threshold. It consists of three main logical blocks:

- **1.1 Message Reception:** Uses a Kafka Trigger node to subscribe to a specified Kafka topic and parse incoming JSON messages.
- **1.2 Conditional Evaluation:** An IF node checks if the temperature value in the incoming message is greater than 50.
- **1.3 Notification Sending:** Based on the IF condition, either sends an SMS alert via Vonage or passes the flow to a NoOp node for messages that do not meet the condition.

---

### 2. Block-by-Block Analysis

#### 2.1 Message Reception

- **Overview:**  
  This block establishes a connection to a Kafka topic named "topic_test" and triggers the workflow whenever a new message arrives. The message is parsed as JSON to extract the temperature value.

- **Nodes Involved:**  
  - Kafka Trigger

- **Node Details:**  
  - **Kafka Trigger**  
    - Type: Kafka Trigger (event-based message listener)  
    - Configuration:  
      - Topic: `topic_test`  
      - Group ID: `n8n` (consumer group identifier for Kafka)  
      - Option enabled: JSON parse message (`jsonParseMessage: true`) which automatically converts incoming message payloads from string to JSON objects.  
    - Key Expressions/Variables:  
      - Incoming message accessible via `{{$json["message"]}}`  
      - Specifically, temperature value accessed as `{{$json["message"]["temp"]}}`  
    - Input Connections: None (trigger node)  
    - Output Connections: Outputs data to the IF node  
    - Version: 1  
    - Potential Failure Types:  
      - Kafka connection/authentication errors  
      - Topic not found or misconfigured  
      - Malformed JSON message parsing errors  
      - Consumer group conflicts  
    - Sub-workflow: None

#### 2.2 Conditional Evaluation

- **Overview:**  
  This block evaluates the temperature value extracted from the Kafka message to determine if it exceeds 50. It routes the workflow accordingly.

- **Nodes Involved:**  
  - IF

- **Node Details:**  
  - **IF**  
    - Type: IF node (conditional branching)  
    - Configuration:  
      - Condition Type: Number comparison  
      - Condition: Check if `{{$node["Kafka Trigger"].json["message"]["temp"]}} > 50`  
    - Input Connections: Receives data from Kafka Trigger  
    - Output Connections:  
      - True branch: Connects to Vonage node (sends SMS alert)  
      - False branch: Connects to NoOp node (no operation)  
    - Version: 1  
    - Potential Failure Types:  
      - Expression evaluation errors if the `temp` field is missing or not numeric  
      - Null or undefined values causing runtime errors  
    - Sub-workflow: None

#### 2.3 Notification Sending

- **Overview:**  
  This block handles sending SMS notifications via Vonage if the temperature condition is met; otherwise, it simply continues the workflow with a NoOp node.

- **Nodes Involved:**  
  - Vonage  
  - NoOp

- **Node Details:**  
  - **Vonage**  
    - Type: Vonage node (SMS sending)  
    - Configuration:  
      - From: `"Vonage APIs"` (sender ID displayed in the SMS)  
      - Message: Dynamic text:  
        ```
        Alert!
        The value of temp is {{$node["Kafka Trigger"].json["message"]["temp"]}}.
        ```  
      - Additional Fields: None set  
    - Input Connections: True branch output of IF node  
    - Output Connections: None  
    - Credentials: Requires Vonage API credentials configured in n8n  
    - Version: 1  
    - Potential Failure Types:  
      - Authentication failures with Vonage API  
      - SMS sending limits or message formatting errors  
      - Network timeouts or API errors  
    - Sub-workflow: None

  - **NoOp**  
    - Type: No Operation node (used to gracefully handle unneeded branches)  
    - Configuration: None  
    - Input Connections: False branch output of IF node  
    - Output Connections: None  
    - Version: 1  
    - Potential Failure Types: None (does nothing)  
    - Sub-workflow: None

---

### 3. Summary Table

| Node Name     | Node Type           | Functional Role                     | Input Node(s)   | Output Node(s) | Sticky Note                                                                                          |
|---------------|---------------------|-----------------------------------|-----------------|----------------|----------------------------------------------------------------------------------------------------|
| Kafka Trigger | Kafka Trigger       | Receive and parse Kafka messages  | -               | IF             |                                                                                                    |
| IF            | IF                  | Evaluate temperature condition    | Kafka Trigger   | Vonage, NoOp   |                                                                                                    |
| Vonage        | Vonage              | Send SMS alert if condition true  | IF (true)       | -              |                                                                                                    |
| NoOp          | No Operation (NoOp) | Handle false condition gracefully | IF (false)      | -              |                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Kafka Trigger node:**  
   - Type: Kafka Trigger  
   - Configure Credentials: Select or create Kafka credential with connection details (brokers, authentication)  
   - Set Topic: `topic_test`  
   - Set Group ID: `n8n`  
   - Enable `JSON Parse Message` option for automatic JSON parsing  
   - Position on canvas (optional): [490,260]

2. **Create IF node:**  
   - Type: IF  
   - Connect Kafka Trigger node's main output to the IF node's input  
   - Configure condition:  
     - Condition type: Number  
     - Value 1: Expression `{{$node["Kafka Trigger"].json["message"]["temp"]}}`  
     - Operation: Greater than  
     - Value 2: `50`  
   - Position on canvas (optional): [690,260]

3. **Create Vonage node:**  
   - Type: Vonage  
   - Connect IF node's "true" output to Vonage node's input  
   - Configure Credentials: Select or create Vonage API credentials (API key and secret)  
   - Set "From" field: `"Vonage APIs"`  
   - Set "Message" field: Expression:  
     ```
     Alert!
     The value of temp is {{$node["Kafka Trigger"].json["message"]["temp"]}}.
     ```  
   - Position on canvas (optional): [890,160]

4. **Create NoOp node:**  
   - Type: No Operation (NoOp)  
   - Connect IF node's "false" output to NoOp node's input  
   - No additional configuration needed  
   - Position on canvas (optional): [890,360]

5. **Activate and Test:**  
   - Activate the workflow  
   - Ensure Kafka topic `topic_test` is correctly producing messages with the JSON structure containing `message.temp`  
   - Validate SMS sending via Vonage when temp > 50  
   - Monitor workflow execution logs for errors and troubleshoot as needed

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                       |
|-----------------------------------------------------------------------------------------------|------------------------------------------------------|
| Vonage API requires a valid API key and secret. Register at https://www.vonage.com/          | Vonage SMS integration                                |
| Kafka topics must be properly configured and accessible to the configured Kafka credentials. | Kafka setup and permissions                           |
| The workflow assumes that incoming Kafka messages include a JSON object under `message.temp`. | Message schema requirement                            |
| NoOp node is used here to handle the false branch gracefully without errors or further actions.| n8n workflow branching best practice                  |

---

This document provides a complete technical reference for the described n8n workflow, enabling advanced users and automated agents to understand, reproduce, and maintain this Kafka-triggered SMS alert system.
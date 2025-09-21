### **Group Message Fan-Out**

**Problem:**
The architecture can now manage group memberships, but it lacks the mechanism to perform a message "fan-out"—delivering a single message to all members of a group. A naive implementation, where the Messaging Service publishes a unique copy of the message for each group member, would fail to meet our peak read load requirement (NFR1.4). A single message to a 500-person group would generate 500 messages on our message bus, creating a massive traffic amplification that would bottleneck the system.

**Solution:**
We will implement a highly efficient **"Fan-out on Read"** (or "Edge Fan-out") pattern.
1.  **Single Publish:** When the **Messaging Service** processes a group message, it will fetch the full member list from the **Group Service**. It will then publish a **single message** to a dedicated `group_outgoing_messages` topic on the Message Bus. This message will contain the payload *and* the complete list of recipient `user_id`s in its metadata.
2.  **Distributed Fan-out:** Every instance in the **Gateway Service** fleet will subscribe to this `group_outgoing_messages` topic. When an instance consumes a message, it will iterate through the recipient list in the metadata. For each recipient, it checks if that user is currently connected to *itself*. If so, it delivers the message. If not, it simply ignores that user ID.

This design ensures the message is processed and transported through the core system only once. The actual fan-out work is distributed across the entire fleet of Gateway servers at the edge, making the process horizontally scalable.

**Trade-offs:**
*   **Fan-out on Read Strategy:**
    *   **Pros:**
        *   **Extreme Network Efficiency:** Prevents message amplification on the message bus, dramatically reducing internal bandwidth and processing costs. One sent message equals one message on the bus.
        *   **Scalable by Design:** The fan-out workload is spread across the Gateway fleet, which scales directly with the number of concurrent users.
    *   **Cons:**
        *   **Increased Gateway CPU Load:** Every Gateway instance must process every group message to check the recipient list. This increases the baseline CPU usage on the Gateway fleet, which must be accounted for in our auto-scaling configuration.
        *   **Larger Message Payloads:** Messages on the bus will be larger due to the included recipient list. This is a favorable trade-off compared to the massive amplification of sending individual messages.
*   **Alternative Considered (Fan-out on Write):**
    *   Rejected because publishing a message for every single recipient would create a "thundering herd" problem, overwhelming the message bus and failing to meet the peak message delivery NFR.

---

#### **Logical View (C4 Component Diagram)**

The logical components remain the same as in Issue #7. The change is in the data flow and the responsibilities of the existing components, which is best captured by refining the interaction labels.

```mermaid
graph TD
    subgraph "Connect Chat System"
        direction LR

        mobile_app("Mobile App\n[Software System]")
        
        subgraph "Backend"
            direction LR
            gateway("Gateway Service\n[Component]\nPerforms physical fan-out to its connected clients.")
            message_bus("Message Bus\n[Component]")
            
            subgraph "Core & Persistence"
                direction TB
                messaging("Messaging Service\n[Component]\nPerforms logical fan-out by attaching a recipient list.")
                group_svc("Group Service\n[Component]")
                group_db("Group Database\n[Container]")
            end
        end
    end

    %% Group Message Send Flow
    mobile_app -- "1.Sends Group Message" --> gateway
    gateway -- "2.Publishes to Bus" --> message_bus
    message_bus -- "3.Consumed by" --> messaging
    messaging -- "4.Gets Member List" --> group_svc
    group_svc -- "5.Reads Members" --> group_db
    messaging -- "6.Publishes ONE message with ALL recipient IDs" --> message_bus
    message_bus -- "7.ALL Gateways consume message" --> gateway
    gateway -- "8.Delivers to locally connected recipients" --> mobile_app
    
    classDef component fill:#85bbf0,stroke:#333,stroke-width:2px;
    classDef system fill:#4387cc,stroke:#333,stroke-width:2px,color:#fff;
    classDef database fill:#f29100,stroke:#333,stroke-width:2px;
    
    class mobile_app system;
    class gateway,messaging,message_bus,group_svc component;
    class group_db database;
```

#### **Physical View (AWS Deployment Diagram)**

The physical resources remain the same. This issue describes a change in the *logic* within the Fargate tasks and the *data* flowing through the Kafka topics, not a change in the infrastructure itself. The Physical View from Issue #7 is still correct.

```mermaid
graph TD
    subgraph "AWS Cloud"
        subgraph "VPC (Private Subnet)"
            direction TB

            gateway_asg("Gateway Service ASG\n(Fargate Tasks)")
            message_bus("Amazon MSK\n[Managed Kafka Cluster]")

            subgraph "Worker Services"
                direction LR
                messaging_asg("Messaging Service ASG\n(Fargate Tasks)")
                group_asg("Group Service ASG\n(Fargate Tasks)")
            end

            nlb("Internal NLB")
            
            subgraph "Data Stores"
                direction LR
                chat_dynamodb("Amazon DynamoDB\n[Chat History]")
                group_dynamodb("Amazon DynamoDB\n[Group Data]")
            end
        end
    end

    %% Define Interactions
    gateway_asg -- "Pub/Sub" --> message_bus
    message_bus -- "Pub/Sub" --> messaging_asg
    
    messaging_asg -- "Calls via" --> nlb
    nlb -- "Routes to" --> group_asg
    group_asg -- "Reads/Writes" --> group_dynamodb
    
    classDef aws_service fill:#FF9900,stroke:#333,stroke-width:2px;
    classDef container fill:#46a3b3,stroke:#333,stroke-width:2px;
    classDef database fill:#C71585,stroke:#333,stroke-width:2px,color:#fff;
    classDef bus fill:#9B59B6,stroke:#333,stroke-width:2px,color:#fff;

    class nlb aws_service;
    class chat_dynamodb,group_dynamodb database;
    class gateway_asg,messaging_asg,group_asg container;
    class message_bus bus;
```

#### **Component-to-Resource Mapping Table**

| Logical Component | Physical Resource | Rationale |
| :--- | :--- | :--- |
| **Gateway Service** | **AWS Fargate Tasks (Auto Scaling)** | (Updated) Subscribes to group message topics. Now responsible for **physical fan-out**: inspecting the recipient list of each message and delivering it only to clients connected to its specific instance. |
| **Messaging Service**| **AWS Fargate Tasks (Auto Scaling)** | (Updated) Responsible for **logical fan-out**: fetching the group member list from the Group Service and publishing a single message to the message bus containing the full list of recipient IDs. |
| **Group Service**| **AWS Fargate Tasks (Auto Scaling)**| (Unchanged) Provides the critical, low-latency lookup of group member lists. |
| **All other components**| (As previously defined) | (Unchanged) No changes to other components. |

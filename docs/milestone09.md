### **Global Message Ordering**

**Problem:**
In a globally distributed, asynchronous system, we cannot rely on processing time to order messages. Network latency or consumer lag could cause a message sent at `T1` to be processed *after* a message sent at `T2`, violating our strict ordering requirement (NFR4.2). This would result in conversations appearing out of order for users, which is a critical failure. We need an authoritative mechanism to guarantee that messages within any given chat are always stored and retrieved in the correct sequence.

**Solution:**
We will implement a centralized sequence generation pattern to assign a unique, monotonically increasing ID to every message within a specific chat.
1.  A new, highly available **Sequence Generator Service** will be introduced.
2.  When the **Messaging Service** consumes a message from the bus, its *first action* will be to request a new `sequence_id` from the Sequence Generator, passing the `chat_id`.
3.  The Sequence Generator will atomically increment a counter for that `chat_id` and return the new number.
4.  This `sequence_id` is then attached to the message.
5.  The **Chat Database (DynamoDB)** will use this ID as its **sort key**. The table's primary key will be a composite key of `Partition Key: chat_id` and `Sort Key: sequence_id`. This structure guarantees that when we query for a chat's history, the messages are always returned in the correct, immutable order.

**Trade-offs:**
*   **Technology Choice (Centralized Sequence Generator):**
    *   **Pros:**
        *   **Guaranteed Total Order:** Provides an absolute, unambiguous order for messages within a chat, which is the simplest and most robust solution for this requirement.
        *   **Idempotency:** The unique `sequence_id` can be used by the persistence layer to safely handle message retries without creating duplicates.
    *   **Cons:**
        *   **Potential Bottleneck:** The Sequence Generator is a critical, high-throughput service. It must be engineered for extremely low latency and high availability. An outage of this service would halt the processing of new messages.
        *   **Added Latency:** This design introduces a synchronous network call into the message processing path. The performance of this service is therefore critical.
*   **Alternative Considered (Using Timestamps):**
    *   Rejected because server clocks can drift, and network latency can cause out-of-order processing, making timestamps unreliable for strict ordering.
*   **Alternative Considered (Client-Side IDs):**
    *   Rejected because clients are untrusted and their clocks can be wildly inaccurate.

---

#### **Logical View (C4 Component Diagram)**

```mermaid
graph TD
    subgraph "Connect Chat System"
        direction LR

        mobile_app("Mobile App\n[Software System]")
        
        subgraph "Backend"
            direction LR
            gateway("Gateway Service\n[Component]")
            message_bus("Message Bus\n[Component]")
            
            subgraph "Core & Persistence"
                direction TB
                messaging("Messaging Service\n[Component]")
                
                subgraph "Sequencing Layer"
                     seq_svc("Sequence Generator Service\n[Component]")
                     seq_db("Sequence Database\n[Container]")
                end

                subgraph "Persistence Layer"
                    persistence_svc("Persistence Service\n[Component]")
                    chat_db("Chat Database\n[Container]")
                end
            end
        end
    end

    %% Complete Message Flow with Sequencing
    mobile_app -- "1.Sends Message" --> gateway
    gateway -- "2.Publishes to Bus" --> message_bus
    message_bus -- "3.Consumed by" --> messaging
    messaging -- "4.Gets sequence_id" --> seq_svc
    seq_svc -- "5.Increments counter" --> seq_db
    messaging -- "6.Persists Message w/ sequence_id" --> persistence_svc
    persistence_svc -- "7.Writes to DB" --> chat_db
    messaging -- "8.Routes for Real-time Delivery" --> message_bus
    
    classDef component fill:#85bbf0,stroke:#333,stroke-width:2px;
    classDef system fill:#4387cc,stroke:#333,stroke-width:2px,color:#fff;
    classDef database fill:#f29100,stroke:#333,stroke-width:2px;
    
    class mobile_app system;
    class gateway,messaging,message_bus,persistence_svc,seq_svc component;
    class chat_db,seq_db database;
```

#### **Physical View (AWS Deployment Diagram)**

```mermaid
graph TD
    subgraph "AWS Cloud"
        subgraph "VPC (Private Subnet)"
            direction TB

            gateway_asg("Gateway Service ASG\n(Fargate Tasks)")
            message_bus("Amazon MSK\n[Managed Kafka Cluster]")
            messaging_asg("Messaging Service ASG\n(Fargate Tasks)")
            
            nlb("Internal NLB")

            subgraph "Downstream Services"
                direction LR
                subgraph "Sequencing Layer"
                    seq_asg("Sequence Generator Service ASG\n(Fargate Tasks)")
                    seq_dynamodb("Amazon DynamoDB\n[Sequence Counters]")
                end
                subgraph "Persistence Layer"
                    persistence_asg("Persistence Service ASG\n(Fargate Tasks)")
                    chat_dynamodb("Amazon DynamoDB\n[Chat History]")
                end
            end
        end
    end

    %% Define Interactions
    gateway_asg -- "Pub/Sub" --> message_bus
    message_bus -- "Pub/Sub" --> messaging_asg
    
    messaging_asg -- "1.Calls for Sequence ID" --> nlb
    nlb -- "Routes to" --> seq_asg
    seq_asg -- "Atomic Updates" --> seq_dynamodb

    messaging_asg -- "2.Calls to Persist Message" --> nlb
    nlb -- "Routes to" --> persistence_asg
    persistence_asg -- "Writes Items" --> chat_dynamodb
    
    classDef aws_service fill:#FF9900,stroke:#333,stroke-width:2px;
    classDef container fill:#46a3b3,stroke:#333,stroke-width:2px;
    classDef database fill:#C71585,stroke:#333,stroke-width:2px,color:#fff;
    classDef bus fill:#9B59B6,stroke:#333,stroke-width:2px,color:#fff;

    class nlb aws_service;
    class chat_dynamodb,seq_dynamodb database;
    class gateway_asg,messaging_asg,seq_asg,persistence_asg container;
    class message_bus bus;
```

#### **Component-to-Resource Mapping Table**

| Logical Component | Physical Resource | Rationale |
| :--- | :--- | :--- |
| **(New) Sequence Generator Service** | **AWS Fargate Tasks (Auto Scaling)** | **Low-Latency & Scalable:** A simple, highly optimized microservice whose only job is to provide sequence numbers. Running on Fargate allows it to scale out to handle the very high request rate from the Messaging Service fleet. |
| **(New) Sequence Database**| **Amazon DynamoDB Table**| **Strong Consistency & Atomic Operations:** DynamoDB's atomic counters are a perfect fit for this use case. They provide a highly available and durable mechanism for atomically incrementing the sequence number for a given `chat_id`, which is the core requirement for this service. |
| **Chat Database** | **Amazon DynamoDB Global Tables**| (Updated) The schema is now defined with a primary key of `(Partition Key: chat_id, Sort Key: sequence_id)` to enforce and leverage the guaranteed ordering provided by the Sequence Generator. |
| **Messaging Service**| **AWS Fargate Tasks (Auto Scaling)** | (Updated) Now contains the critical logic to call the Sequence Generator *before* calling the Persistence Service, ensuring every message is correctly sequenced before being stored. |

### **Inter-Service Communication & 1-to-1 Routing**

**Problem:**
The current architecture implies a direct, synchronous communication path between the Gateway and Messaging services. This creates tight coupling, where a slowdown or failure in the Messaging Service could block the Gateway Service, jeopardizing its ability to manage thousands of client connections. This synchronous model is not resilient enough to meet our high availability (NFR3.1) and guaranteed delivery (NFR4.1) requirements at scale.

**Solution:**
We will introduce a **Message Bus** to act as the central, asynchronous communication backbone between services, implementing a publish-subscribe (Pub/Sub) pattern.
1.  **Gateway Service (Producer):** Upon receiving a message from a client, the Gateway will publish it to a message bus topic (e.g., `incoming_messages`).
2.  **Messaging Service (Consumer/Producer):** This service will subscribe to the `incoming_messages` topic. After processing a message (e.g., validation, querying the Session Registry), it will publish the message to a topic designated for delivery (e.g., `outgoing_messages`).
3.  **Gateway Service (Consumer):** Each Gateway instance will also subscribe to the `outgoing_messages` topic. It will consume messages and deliver them only to the clients currently connected to it.

This asynchronous model decouples the services, provides a durable buffer for messages, and enhances overall system resilience and scalability.

**Trade-offs:**
*   **Technology Choice (Message Bus - Amazon MSK/Kafka):**
    *   **Pros:**
        *   **Decoupling:** Services can be developed, deployed, and scaled independently.
        *   **Resilience & Durability:** The bus acts as a buffer. If a consumer service fails, messages are retained in the bus and can be processed upon recovery, ensuring no data loss.
        *   **Load Balancing:** The bus naturally distributes the message load among available consumer instances.
    *   **Cons:**
        *   **Increased Latency:** Introduces an additional network hop, which slightly increases end-to-end latency. However, the gains in reliability and scalability are a necessary trade-off.
        *   **Operational Complexity:** Adds a new, critical piece of infrastructure that requires management and monitoring. Using a managed service like Amazon MSK mitigates much of this burden.
*   **Alternative Considered (Direct RPC):**
    *   Rejected due to the tight coupling and lack of fault tolerance. A failure in the `Messaging Service` would cascade directly to the `Gateway Service`, potentially causing widespread disconnections.

---

#### **Logical View (C4 Component Diagram)**

```mermaid
graph LR
    subgraph "Connect Chat System"
        direction LR

        subgraph "Client"
             user("User\n[Person]")
             mobile_app("Mobile App\n[Software System]")
        end
        
        subgraph "Backend"
            direction LR
            gateway("Gateway Service\n[Component]")
            message_bus("Message Bus\n[Component]\nAsynchronous, durable message backbone.")
            
            subgraph "Core Services"
                direction TB
                messaging("Messaging Service\n[Component]")
                registry("Session Registry\n[Component]")
            end
        end
    end

    %% Numbered flow for clarity
    user -- "Uses" --> mobile_app
    mobile_app -- "1.Sends Message" --> gateway
    gateway -- "2.Publishes to 'incoming_messages' topic" --> message_bus
    message_bus -- "3.Consumed by" --> messaging
    messaging -- "4.Looks up recipient session" --> registry
    messaging -- "5.Publishes to 'outgoing_messages' topic" --> message_bus
    message_bus -- "6.Consumed by" --> gateway
    gateway -- "7.Delivers message via WebSocket" --> mobile_app

    classDef component fill:#85bbf0,stroke:#333,stroke-width:2px;
    classDef person fill:#1168bd,stroke:#333,stroke-width:2px,color:#fff;
    classDef system fill:#4387cc,stroke:#333,stroke-width:2px,color:#fff;
    
    class user person;
    class mobile_app system;
    class gateway,messaging,registry,message_bus component;
```

#### **Physical View (AWS Deployment Diagram)**

```mermaid
graph TD
    subgraph "AWS Cloud"
        subgraph "VPC (Virtual Private Cloud)"
            direction TB
            subgraph "Public Subnet"
                alb("Application Load Balancer")
            end

            subgraph "Private Subnet"
                direction TB
                
                gateway_asg("Gateway Service ASG\n(Fargate Tasks)")
                
                message_bus("Amazon MSK\n[Managed Kafka Cluster]")
                
                messaging_asg("Messaging Service ASG\n(Fargate Tasks)")
                
                session_registry("Amazon ElastiCache for Redis\n[Cluster Mode]")
            end
        end
    end
    
    %% Define the numbered message flow
    alb -- "1.Routes WSS Traffic" --> gateway_asg
    gateway_asg -- "2.Publishes to 'incoming_messages' topic" --> message_bus
    message_bus -- "3.Messaging Service consumes" --> messaging_asg
    messaging_asg -- "4.Looks up recipient session" --> session_registry
    messaging_asg -- "5.Publishes to 'outgoing_messages' topic" --> message_bus
    message_bus -- "6.Gateway Service consumes" --> gateway_asg
    
    classDef aws_service fill:#FF9900,stroke:#333,stroke-width:2px;
    classDef container fill:#46a3b3,stroke:#333,stroke-width:2px;
    classDef cache fill:#D64541,stroke:#333,stroke-width:2px,color:#fff;
    classDef bus fill:#9B59B6,stroke:#333,stroke-width:2px,color:#fff;

    class alb aws_service;
    class message_bus bus;
    class session_registry cache;
    class gateway_asg,messaging_asg container;
```

#### **Component-to-Resource Mapping Table**

| Logical Component | Physical Resource | Rationale |
| :--- | :--- | :--- |
| **Gateway Service** | **AWS Fargate Tasks (Auto Scaling)** | (Unchanged) Serverless compute for managing stateful WebSocket connections. Acts as a Producer and Consumer to the message bus. |
| **Messaging Service**| **AWS Fargate Tasks (Auto Scaling)** | (Unchanged) Stateless, serverless compute for core logic. Acts as a Consumer and Producer to the message bus. |
| **Session Registry**| **Amazon ElastiCache for Redis (Cluster Mode)** | (Unchanged) Provides a high-performance, low-latency, and scalable in-memory store for session mapping. |
| **(New) Message Bus**| **Amazon MSK (Managed Streaming for Apache Kafka)** | **Durability & Scalability:** MSK provides a fully managed, highly available Kafka cluster. Kafka is the industry standard for high-throughput, persistent message streaming, making it ideal for decoupling our services and buffering messages to guarantee delivery, even during service downtime. |

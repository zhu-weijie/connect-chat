#### **Overall Logical View (C4 Component Diagram)**

This diagram represents the complete logical architecture of the Connect Chat service, showing all components and their primary responsibilities.

```mermaid
graph LR
    subgraph "Connect Chat System"
        direction LR

        mobile_app("Mobile App\n[Software System]")
        
        subgraph "Backend"
            direction LR
            
            gateway("Gateway Service\n[Component]\nManages client connections.")
            
            message_bus("Message Bus\n[Component]\nAsynchronous message backbone.")
            
            subgraph "Core Services"
                direction TB
                messaging("Messaging Service")
                group_svc("Group Service")
                seq_svc("Sequence Generator")
                notification_svc("Notification Service")
            end

            subgraph "Data Stores"
                direction TB
                session_registry("Session Registry (Redis)")
                chat_db("Chat Database (DynamoDB)")
                group_db("Group Database (DynamoDB)")
                seq_db("Sequence Database (DynamoDB)")
                offline_store("Offline Store (Redis)")
            end
        end
    end

    %% High-level interactions
    mobile_app -- "API (HTTPS)\nReal-time (WSS)" --> gateway
    
    gateway -- "Pub/Sub" --> message_bus
    message_bus -- "Pub/Sub" --> messaging
    message_bus -- "Pub/Sub" --> notification_svc

    messaging -- "Uses" --> session_registry
    messaging -- "Uses" --> chat_db
    messaging -- "Uses" --> group_svc
    messaging -- "Uses" --> seq_svc
    
    group_svc -- "Uses" --> group_db
    seq_svc -- "Uses" --> seq_db
    notification_svc -- "Uses" --> offline_store


    classDef component fill:#85bbf0,stroke:#333,stroke-width:2px;
    classDef system fill:#4387cc,stroke:#333,stroke-width:2px,color:#fff;
    classDef database fill:#f29100,stroke:#333,stroke-width:2px;
    
    class mobile_app system;
    class gateway,messaging,message_bus,group_svc,seq_svc,notification_svc component;
    class chat_db,group_db,session_registry,seq_db,offline_store database;
```

#### **Overall Physical View (AWS Deployment Diagram)**

This diagram represents the complete, multi-region, highly available physical deployment architecture for the Connect Chat service.

```mermaid
graph TD
    user("User")

    subgraph "Global"
        route53("Amazon Route 53\n[Latency-based Routing]")
    end
    
    user --> route53

    subgraph "AWS Region A (Active)"
        direction TB
        subgraph "VPC A"
            alb_a("ALB") --> gateway_a("Gateway Service ASG")
            
            subgraph "Private Subnet A"
                gateway_a
                msk_a("Amazon MSK")
                services_a("Core Services ASGs\n(Messaging, Group, Sequence, Notification via NLB)")
                datastores_a("Managed Databases\n(DynamoDB Global Tables, ElastiCache)")
            end
        end
    end

    subgraph "AWS Region B (Active)"
        direction TB
         subgraph "VPC B"
            alb_b("ALB") --> gateway_b("Gateway Service ASG")
            
            subgraph "Private Subnet B"
                gateway_b
                msk_b("Amazon MSK")
                services_b("Core Services ASGs\n(Messaging, Group, Sequence, Notification via NLB)")
                datastores_b("Managed Databases\n(DynamoDB Global Tables, ElastiCache)")
            end
        end
    end

    route53 -- "Routes User to Nearest Region" --> alb_a & alb_b

    %% Cross-Region Replication
    datastores_a <-->|"Data Replication"| datastores_b
    msk_a <-->|"Cross-Cluster Replication"| msk_b
    
    classDef aws_service fill:#FF9900,stroke:#333,stroke-width:2px;
    classDef container fill:#46a3b3,stroke:#333,stroke-width:2px;
    classDef database fill:#C71585,stroke:#333,stroke-width:2px,color:#fff;
    classDef bus fill:#9B59B6,stroke:#333,stroke-width:2px,color:#fff;

    class route53,alb_a,alb_b aws_service;
    class msk_a,msk_b bus;
    class datastores_a,datastores_b database;
    class gateway_a,gateway_b,services_a,services_b container;
```

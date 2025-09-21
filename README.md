# Connect Chat

## Logical View

### Milestone 01: System Scaffolding & Real-time Protocol

```mermaid
graph TD
    user("User\n[Person]")

    subgraph "Connect Chat System"
        direction LR
        
        mobile_app("Mobile App\n[Software System]")
        
        subgraph "Backend"
            direction TB
            gateway("Gateway Service\n[Component]\nManages WebSocket connections and user sessions.")
            messaging("Messaging Service\n[Component]\nHandles message processing and core chat logic.")
        end
    end
    
    user -- "Uses" --> mobile_app
    
    mobile_app -- "Sends Message\n(WSS)" --> gateway
    gateway -- "Delivers Message\n(WSS)" --> mobile_app
    
    gateway -- "Forwards Message for Processing\n(RPC/Message Bus)" --> messaging
    messaging -- "Routes Message for Delivery\n(RPC/Message Bus)" --> gateway

    classDef component fill:#85bbf0,stroke:#333,stroke-width:2px;
    classDef person fill:#1168bd,stroke:#333,stroke-width:2px,color:#fff;
    classDef system fill:#4387cc,stroke:#333,stroke-width:2px,color:#fff;
    
    class user person;
    class mobile_app system;
    class gateway,messaging component;
```

### Milestone 02: Session Registry & Service Discovery

```mermaid
graph TD
    subgraph "Connect Chat System"
        direction TB

        user("User\n[Person]")
        mobile_app("Mobile App\n[Software System]")
        
        subgraph "Backend"
            direction LR
            gateway("Gateway Service\n[Component]\nManages WebSocket connections and user sessions.")
            
            subgraph "Core Services"
                direction TB
                messaging("Messaging Service\n[Component]\nHandles message processing and core chat logic.")
                registry("Session Registry\n[Component]\nStores user_id -> gateway_instance mapping.")
            end
        end
    end
    
    user -- "Uses" --> mobile_app

    %% Numbered flow for clarity
    mobile_app -- "1.Establishes WebSocket Connection" --> gateway
    gateway -- "2.Registers Session\n(user_id, gateway_addr, TTL)" --> registry
    gateway -- "3.Forwards inbound message" --> messaging
    messaging -- "4.Looks up recipient's gateway address" --> registry
    messaging -- "5.Routes message to target gateway" --> gateway
    gateway -- "6.Delivers message to recipient" --> mobile_app

    classDef component fill:#85bbf0,stroke:#333,stroke-width:2px;
    classDef person fill:#1168bd,stroke:#333,stroke-width:2px,color:#fff;
    classDef system fill:#4387cc,stroke:#333,stroke-width:2px,color:#fff;
    
    class user person;
    class mobile_app system;
    class gateway,messaging,registry component;
```

### Milestone 03: Inter-Service Communication & 1-to-1 Routing

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

### Milestone 04: Chat History Persistence

```mermaid
graph LR
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
                persistence_svc("Persistence Service\n[Component]")
                chat_db("Chat Database\n[Container]")
            end
        end
    end

    %% Real-time Send Flow (Flow 'a')
    mobile_app -- "1a. Sends Message (WSS)" --> gateway
    gateway -- "2a. Publishes" --> message_bus
    message_bus -- "3a. Consumed by" --> messaging
    messaging -- "4a. Persists Message" --> persistence_svc
    persistence_svc -- "5a. Writes to DB" --> chat_db
    messaging -- "6a. Routes for Real-time Delivery" --> message_bus
    message_bus -- "7a. Consumed by" --> gateway
    gateway -- "8a. Delivers to Client (WSS)" --> mobile_app

    %% History Fetch Flow (Flow 'b')
    mobile_app -- "1b. Requests History (HTTPS)" --> gateway
    gateway -- "2b. Forwards Request" --> messaging
    messaging -- "3b. Forwards Request" --> persistence_svc
    persistence_svc -- "4b. Reads from DB" --> chat_db
    
    classDef component fill:#85bbf0,stroke:#333,stroke-width:2px;
    classDef system fill:#4387cc,stroke:#333,stroke-width:2px,color:#fff;
    classDef database fill:#f29100,stroke:#333,stroke-width:2px;
    
    class mobile_app system;
    class gateway,messaging,message_bus,persistence_svc component;
    class chat_db database;
```

### Milestone 05: Offline Message Delivery System

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
                notification_svc("Notification Service\n[Component]")
                
                subgraph "Data Stores"
                    direction LR
                    offline_store("Offline Message Store\n[Container]")
                    chat_db("Chat Database\n[Container]")
                end
            end
        end
    end

    %% User connects and gets offline messages
    mobile_app -- "1.Connects" --> gateway
    gateway -- "2.Publishes 'user_connected' event" --> message_bus
    message_bus -- "3.Consumed by" --> notification_svc
    notification_svc -- "4.Fetches messages" --> offline_store
    notification_svc -- "5.Publishes offline messages for delivery" --> message_bus
    message_bus -- "6.Consumed by" --> gateway
    gateway -- "7.Delivers to Client" --> mobile_app

    %% When a message is sent to an offline user
    messaging -- "If recipient offline, writes message" --> offline_store

    classDef component fill:#85bbf0,stroke:#333,stroke-width:2px;
    classDef system fill:#4387cc,stroke:#333,stroke-width:2px,color:#fff;
    classDef database fill:#f29100,stroke:#333,stroke-width:2px;
    
    class mobile_app system;
    class gateway,messaging,message_bus,notification_svc component;
    class chat_db,offline_store database;
```

### Milestone 06: Horizontal Scalability & High Availability

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
                notification_svc("Notification Service\n[Component]")
                
                subgraph "Data Stores"
                    direction LR
                    offline_store("Offline Message Store\n[Container]")
                    chat_db("Chat Database\n[Container]")
                end
            end
        end
    end

    %% User connects and gets offline messages
    mobile_app -- "1.Connects" --> gateway
    gateway -- "2.Publishes 'user_connected' event" --> message_bus
    message_bus -- "3.Consumed by" --> notification_svc
    notification_svc -- "4.Fetches messages" --> offline_store
    notification_svc -- "5.Publishes offline messages for delivery" --> message_bus
    message_bus -- "6.Consumed by" --> gateway
    gateway -- "7.Delivers to Client" --> mobile_app

    %% When a message is sent to an offline user
    messaging -- "If recipient offline, writes message" --> offline_store

    classDef component fill:#85bbf0,stroke:#333,stroke-width:2px;
    classDef system fill:#4387cc,stroke:#333,stroke-width:2px,color:#fff;
    classDef database fill:#f29100,stroke:#333,stroke-width:2px;
    
    class mobile_app system;
    class gateway,messaging,message_bus,notification_svc component;
    class chat_db,offline_store database;
```

### Milestone 07: Group Management Service

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
                group_svc("Group Service\n[Component]")
                subgraph "Data Stores"
                    direction LR
                    group_db("Group Database\n[Container]")
                    chat_db("Chat Database\n[Container]")
                end
            end
        end
    end

    %% Group Message Send Flow
    mobile_app -- "1.Sends Group Message" --> gateway
    gateway -- "2.Publishes to Bus" --> message_bus
    message_bus -- "3.Consumed by" --> messaging
    messaging -- "4.Gets Member List" --> group_svc
    group_svc -- "5.Reads Members" --> group_db
    messaging -- "6.Routes Message for Fan-Out" --> message_bus
    
    classDef component fill:#85bbf0,stroke:#333,stroke-width:2px;
    classDef system fill:#4387cc,stroke:#333,stroke-width:2px,color:#fff;
    classDef database fill:#f29100,stroke:#333,stroke-width:2px;
    
    class mobile_app system;
    class gateway,messaging,message_bus,group_svc component;
    class chat_db,group_db database;
```

### Milestone 08: Group Message Fan-Out

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

### Milestone 09: Global Message Ordering

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

### Overall Logical View

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

## Physical View

### Milestone 01: System Scaffolding & Real-time Protocol

```mermaid
graph TD
    subgraph "Internet"
        user("User")
    end

    subgraph "AWS Cloud"
        subgraph "VPC (Virtual Private Cloud)"
            direction TB
            subgraph "Public Subnet"
                alb("Application Load Balancer\n[Layer 7 LB]")
            end

            subgraph "Private Subnet"
                direction TB
                
                subgraph "Gateway Service Auto Scaling Group"
                    gateway_task("Fargate Task\n[Container: Gateway]")
                end

                nlb("Internal Network Load Balancer\n[Layer 4 LB]")

                subgraph "Messaging Service Auto Scaling Group"
                    messaging_task("Fargate Task\n[Container: Messaging]")
                end
            end
        end
        route53("Route 53\n[DNS]")
    end
    
    user -- "connect.chatapp.com\n(HTTPS/WSS)" --> route53
    route53 --> alb
    alb -- "Forwards WSS Traffic\n(Sticky Sessions)" --> gateway_task
    gateway_task -- "Routes via Internal DNS" --> nlb
    nlb -- "Distributes load" --> messaging_task

    classDef aws_service fill:#FF9900,stroke:#333,stroke-width:2px;
    classDef container fill:#46a3b3,stroke:#333,stroke-width:2px;

    class route53,alb,nlb aws_service;
    class gateway_task,messaging_task container;
```

### Milestone 02: Session Registry & Service Discovery

```mermaid
graph TD
    subgraph "Internet"
        user("User")
    end

    subgraph "AWS Cloud"
        subgraph "VPC (Virtual Private Cloud)"
            direction TB
            subgraph "Public Subnet"
                alb("Application Load Balancer")
            end

            subgraph "Private Subnet"
                direction TB
                
                subgraph "Gateway Service Auto Scaling Group"
                    gt1("Fargate Task 1")
                    gt2("...")
                    gtN("Fargate Task N")
                end

                subgraph "Session Registry"
                    redis("Amazon ElastiCache for Redis\n[Cluster Mode]")
                end

                nlb("Internal NLB")

                subgraph "Messaging Service Auto Scaling Group"
                    mt1("Fargate Task 1")
                    mt2("...")
                    mtN("Fargate Task N")
                end
            end
        end
        route53("Route 53")
    end
    
    user -- "connect.chatapp.com\n(HTTPS/WSS)" --> route53
    route53 --> alb
    alb -- "Routes Traffic" --> gt1 & gt2 & gtN
    
    gt1 & gt2 & gtN -- "Writes Session (TCP)" --> redis
    gt1 & gt2 & gtN -- "Routes via Internal DNS" --> nlb
    
    nlb -- "Distributes load" --> mt1 & mt2 & mtN
    mt1 & mt2 & mtN -- "Reads Session (TCP)" --> redis

    classDef aws_service fill:#FF9900,stroke:#333,stroke-width:2px;
    classDef container fill:#46a3b3,stroke:#333,stroke-width:2px;
    classDef cache fill:#D64541,stroke:#333,stroke-width:2px,color:#fff;

    class route53,alb,nlb aws_service;
    class redis cache;
    class gt1,gt2,gtN,mt1,mt2,mtN container;
```

### Milestone 03: Inter-Service Communication & 1-to-1 Routing

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

### Milestone 04: Chat History Persistence

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
                
                subgraph "Persistence Layer"
                    persistence_asg("Persistence Service ASG\n(Fargate Tasks)")
                    dynamodb("Amazon DynamoDB Table\n[NoSQL Database]")
                end
            end
        end
    end
    
    alb --> gateway_asg
    gateway_asg -- "Pub/Sub" --> message_bus
    message_bus -- "Pub/Sub" --> messaging_asg
    
    messaging_asg -- "Calls via Internal LB (not shown)" --> persistence_asg
    persistence_asg -- "Writes/Reads Items" --> dynamodb

    classDef aws_service fill:#FF9900,stroke:#333,stroke-width:2px;
    classDef container fill:#46a3b3,stroke:#333,stroke-width:2px;
    classDef database fill:#C71585,stroke:#333,stroke-width:2px,color:#fff;
    classDef bus fill:#9B59B6,stroke:#333,stroke-width:2px,color:#fff;

    class alb aws_service;
    class message_bus bus;
    class dynamodb database;
    class gateway_asg,messaging_asg,persistence_asg container;
```

### Milestone 05: Offline Message Delivery System

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
                
                subgraph "Worker Services"
                    direction LR
                    messaging_asg("Messaging Service ASG\n(Fargate Tasks)")
                    notification_asg("Notification Service ASG\n(Fargate Tasks)")
                end

                subgraph "Data Stores"
                    direction LR
                    offline_redis("Amazon ElastiCache for Redis\n[Offline Store]")
                    dynamodb("Amazon DynamoDB\n[Chat History]")
                end
            end
        end
    end

    %% Offline Delivery Flow
    alb -- "1.User Connects" --> gateway_asg
    gateway_asg -- "2.Publishes 'user_connected' event" --> message_bus
    message_bus -- "3.Consumed by" --> notification_asg
    notification_asg -- "4.Fetches messages from" --> offline_redis
    notification_asg -- "5.Publishes messages to" --> message_bus
    message_bus -- "6.Consumed by" --> gateway_asg
    
    %% Separate interaction: Storing an offline message
    messaging_asg -- "Writes to" --> offline_redis


    classDef aws_service fill:#FF9900,stroke:#333,stroke-width:2px;
    classDef container fill:#46a3b3,stroke:#333,stroke-width:2px;
    classDef database fill:#C71585,stroke:#333,stroke-width:2px,color:#fff;
    classDef bus fill:#9B59B6,stroke:#333,stroke-width:2px,color:#fff;
    classDef cache fill:#D64541,stroke:#333,stroke-width:2px,color:#fff;

    class alb aws_service;
    class message_bus bus;
    class dynamodb database;
    class offline_redis cache;
    class gateway_asg,messaging_asg,notification_asg container;
```

### Milestone 06: Horizontal Scalability & High Availability

```mermaid
graph TD
    user("User")

    subgraph "Global"
        route53("Amazon Route 53\n[Latency-based Routing]")
    end
    
    user --> route53

    subgraph "AWS Region A (e.g., us-east-1)"
        direction TB
        subgraph "VPC A"
            alb_a("ALB") --> gateway_a("Gateway ASG")
            gateway_a -- Pub/Sub --> msk_a("Amazon MSK")
            msk_a -- Pub/Sub --> messaging_a("Messaging ASG")
            messaging_a --> persistence_a("Persistence ASG")
            persistence_a --> dynamodb_a("DynamoDB")
        end
    end

    subgraph "AWS Region B (e.g., eu-west-1)"
        direction TB
        subgraph "VPC B"
            alb_b("ALB") --> gateway_b("Gateway ASG")
            gateway_b -- Pub/Sub --> msk_b("Amazon MSK")
            msk_b -- Pub/Sub --> messaging_b("Messaging ASG")
            messaging_b --> persistence_b("Persistence ASG")
            persistence_b --> dynamodb_b("DynamoDB")
        end
    end

    route53 -- "Routes User to Nearest Region" --> alb_a & alb_b

    %% Cross-Region Replication
    dynamodb_a <-->|"Global Table Replication"| dynamodb_b
    msk_a <-->|"Cross-Cluster Replication"| msk_b

    classDef aws_service fill:#FF9900,stroke:#333,stroke-width:2px;
    classDef container fill:#46a3b3,stroke:#333,stroke-width:2px;
    classDef database fill:#C71585,stroke:#333,stroke-width:2px,color:#fff;
    classDef bus fill:#9B59B6,stroke:#333,stroke-width:2px,color:#fff;

    class route53,alb_a,alb_b aws_service;
    class msk_a,msk_b bus;
    class dynamodb_a,dynamodb_b database;
    class gateway_a,gateway_b,messaging_a,messaging_b,persistence_a,persistence_b container;
```

### Milestone 07: Group Management Service

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

### Milestone 08: Group Message Fan-Out

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

### Milestone 09: Global Message Ordering

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

### Overall Physical View

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

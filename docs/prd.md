### **PRD: Project Connect**

*   **Version:** v1.0.0
*   **Date:** 2025-09-20

---

**1. Vision & Business Goals**

To build a globally-distributed, highly reliable chat service engineered to scale to 1 billion users. The system's success will be measured by its ability to provide a real-time communication experience defined by low-latency (<500ms P99) and high-availability (99.99%).

**2. Functional Requirements (FR)**

**FR1: One-to-One Messaging**
*   **FR1.1:** Users can send and receive text-based messages in a private, one-to-one chat session.
*   **FR1.2:** Chat history for each conversation must be persisted and viewable by the users involved.

**FR2: Group Messaging**
*   **FR2.1:** Users can create chat groups and invite other users to join.
*   **FR2.2:** Members of a group can send and receive text-based messages, visible to all other members.
*   **FR2.3:** The maximum number of members in a single group is **500**.
*   **FR2.4:** Basic group management is required (e.g., adding/removing members).

**FR3: Core Messaging Features**
*   **FR3.1: Real-time Delivery:** Messages sent to online users must be delivered instantly.
*   **FR3.2: Offline Storage & Delivery:** Messages sent to offline users must be stored and delivered immediately upon their next connection. Stored offline messages have a TTL (Time-To-Live) of 30 days.
*   **FR3.3: Read Receipts:** The system must support read receipts (e.g., "delivered," "read") that are synchronized across all of the recipient's devices.

**FR4: User Presence**
*   **FR4.1:** Users can see the `online` or `offline` status of their contacts.
*   **FR4.2:** A real-time `typing...` indicator must be displayed to users in a conversation when another user is actively composing a message.

**3. Non-Functional Requirements (NFR)**

**NFR1: Scalability & Performance**
*   **NFR1.1: User Base:** The system must support **1 billion total registered users**.
*   **NFR1.2: Concurrency:** The system must handle a peak load of **50 million concurrent online users**.
*   **NFR1.3: Peak Write Load:** The system must handle **10 million messages sent per second** during peak hours.
*   **NFR1.4: Peak Read Load (Fan-out):** The system must handle **100 million message deliveries per second** during peak hours (assuming an average fan-out of 10 for group chats and notifications).

**NFR2: Latency**
*   **NFR2.1:** The **P99 latency** for real-time message delivery (from sender's "send" action to receiver's "render" action) must be **< 500ms**.
*   **NFR2.2:** Presence status updates (`online`, `typing...`) must appear within **< 2 seconds**.

**NFR3: Availability**
*   **NFR3.1:** The core messaging service must achieve **99.99% uptime** (approx. 52 minutes of downtime per year).
*   **NFR3.2:** The system must be resilient to regional data center failures.

**NFR4: Durability & Consistency**
*   **NFR4.1: Message Durability:** The system must guarantee **"at-least-once" delivery**. No messages should be lost after the sender's client receives a server-side confirmation (`message_sent_ack`).
*   **NFR4.2: Message Ordering:** Within any single chat session (both 1:1 and group), messages must be delivered to all recipients in the exact sequence they were sent.

**NFR5: Security**
*   **NFR5.1: Data in Transit:** All client-server communication must be encrypted using **TLS 1.2+**.
*   **NFR5.2: Data at Rest:** All persisted user data, including offline messages and chat history, must be encrypted (e.g., AES-256).

**4. Out of Scope**

*   Voice/video calls and media sharing (images, videos, files).
*   End-to-end encryption (E2EE).
*   Social features (e.g., user profiles, status updates/stories).
*   Advanced message interactions (e.g., editing, deleting, quoting messages).

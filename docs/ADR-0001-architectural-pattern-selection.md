# ADR-0001: Architecture Pattern Selection for Online Ticket Flash Sale System

## Context
The Online Ticket Flash Sale System is a public-facing event ticketing platform used by ticket buyers, venue administrators, system operators, and event organizers. The system must support event browsing, real-time ticket inventory queries, flash-sale order placement, payment integration, order lifecycle management, automated unpaid stock releases, system monitoring, and audit logging.

The system has several architecturally significant requirements:
- It must handle extreme traffic spikes (high QPS/TPS) during ticket sale ignition and deadlines;
- It must remain highly available during active ticket sale periods;
- It must enforce absolute zero over-selling (No Over-selling) to maintain data consistency under massive concurrency;
- It must protect sensitive user authentication, payment, and transaction data;
- It must keep all purchase attempts, order states, and administrative actions auditable;
- It must integrate reliably with third-party payment gateways;
- It must provide rapid, low-latency responses (under 50 ms for request ingestion) without locking persistent storage;
- It must support dynamic flash-sale events with varying seat rules, quota limits, and pricing structures.

A single architectural style must therefore balance extreme scalability, low latency, maintainability, reliability, security, auditability, and implementation complexity.

## Decision
We will use Event-Driven Architecture (EDA) as the Primary Architectural Paradigm, combined with CQRS (Command Query Responsibility Segregation) for Read/Write Path Separation.

The system will be divided into clear components based on business capabilities. User-facing query workflows will use synchronous API and caching communication when an immediate response is required. Background order processing and stock mutations will use asynchronous event-driven communication.

Main system components include:
- Client Web/Mobile Application;
- API Gateway & Dynamic Token Validation Service;
- User Identity & Rate Limiting Service;
- Ticket Inquiry & Read-Side Cache Service (CQRS Read Path);
- Redis Lua Pre-deduction Engine (CQRS Write Path - Producer);
- Kafka Event Broker / Message Queue;
- Asynchronous Order Processing Worker Cluster (EDA Consumer);
- Relational Database (Primary-Secondary MySQL with Read/Write Splitting);
- Delay Queue / Unpaid Order Timeout Service;
- Notification Service (WebSocket Push);
- Audit Logging & Monitoring Service;
- Third-Party Payment Gateway Integration.

Synchronous communication will be used for actions such as user authentication, event browsing, real-time ticket availability checks (Redis cache hits), and payment gateway status synchronization.

Asynchronous communication will be used for actions such as queueing purchase commands, Order Persistence, unpaid stock release timers, WebSocket status notifications, audit logging, and metric tracking.

## Y-Statement
In the context of building an Online Ticket Flash Sale System as a high-concurrency event ticketing platform,
facing extreme traffic spikes, strict zero over-selling constraints, auditability requirements, third-party payment integration, and low-latency user feedback targets,
we decided on an Event-Driven Architecture (EDA) combined with CQRS (Command Query Responsibility Segregation),
and rejected a simple monolithic architecture, a traditional layered architecture, a full microservices architecture, and a pure event-driven architecture,
to achieve extreme scalability, ultra-low latency, high availability, zero over-selling safety, auditability, and reliable background peak shaving,
accepting higher system complexity than a monolith and the need to manage eventual consistency and asynchronous workflows,
because the system needs clear separation of read/write responsibilities and scalable queueing workflows without the full operational burden and network latency overhead of complete microservices.

## Considered Options

### Option 1: Monolithic Architecture
A monolithic architecture would place all ticket inquiry, order placement, payment processing, and audit functions into a single deployable application database unit.

#### Advantages
- Simple initial development and local testing.
- Single database transaction guarantees simple immediate consistency.
- Lower operational complexity and deployment overhead for small-scale applications.

#### Disadvantages
- Cannot scale ticket purchase functions independently during high-concurrency flash sale peaks.
- High database lock contention under simultaneous write requests leads to thread pool exhaustion and system failure.
- Third-party payment delays or database bottlenecks directly degrade the entire application.
- High risk of inventory over-selling under unhandled race conditions.

#### Evaluation
A monolith is entirely unsuitable for flash-sale scenarios because synchronous database writes cannot handle extreme burst traffic spikes.

### Option 2: Traditional Layered Architecture
A layered architecture strictly organizes code vertically into Presentation Layer, Business Logic Layer, and Data Access Layer with synchronous calls.

#### Advantages
- Well-known standard structure with clear separation of code concerns.
- Easy for development teams to organize.

#### Disadvantages
- End-to-end synchronous call chains (`Client -> Gateway -> Service -> Database`) block threads while waiting for DB IO.
- Thread starvation occurs immediately when thousands of requests hit the database simultaneously.
- Does not separate high-volume read traffic from high-concurrency write traffic.

#### Evaluation
Layered architecture fails under flash-sale workloads because synchronous vertical execution blocks system resources during demand spikes.

### Option 3: Full Microservices Architecture
A full microservices architecture splits each business capability into an independently deployable service, each managing its own isolated database.

#### Advantages
- Independent scalability and deployment for each business service.
- Clear domain ownership across large engineering teams.

#### Disadvantages
- High operational complexity requiring Kubernetes, service meshes, and complex CI/CD pipelines.
- Distributed transactions (e.g., SAGA/TCC) across microservice databases introduce latency and implementation risk.
- Increased network hop latency is undesirable for sub-50ms flash sale responses.

#### Evaluation
Full microservices introduce excessive operational burden and distributed transaction complexity without solving the fundamental high-concurrency database peak-shaving problem.

### Option 4: Pure Event-Driven Architecture
A pure event-driven architecture makes every interaction, including ticket browsing and login, asynchronous via event buses.

#### Advantages
- Maximum decoupling and asynchronous execution.
- Excellent for background execution, audit logging, and audit history.

#### Disadvantages
- Poor user experience for synchronous actions (e.g., browsing tickets or logging in).
- Higher complexity for user-facing request-response workflows.
- Difficult to guarantee immediate read consistency where expected.

#### Evaluation
Pure event-driven models are unsuitable because ticket queries require immediate synchronous responses via cache.

### Option 5: Event-Driven Architecture (EDA) + CQRS Pattern
This option separates the system into a high-throughput CQRS Read Path (Redis cache) and an EDA Write Path (Redis Lua pre-deduction + Kafka queue + asynchronous workers).

#### Advantages
- **CQRS Read Path**: 99% of ticket browsing hits Redis/CDN directly, achieving $100,000+\text{ QPS}$ without touching the relational database.
- **EDA Write Path**: Flash-sale purchase commands are ingested in memory via Lua scripts and pushed to Kafka in $<50\text{ ms}$, shaving traffic peaks.
- **Zero Over-selling Guarantee**: Redis Lua scripts ensure atomic stock deduction; Kafka sequential partition consumption prevents database race conditions.
- **Fault Isolation**: Database slowness does not block request ingestion in Kafka.

#### Disadvantages
- Requires managing eventual consistency between queue completion and user UI updates.
- Higher complexity than a monolith, requiring event handling, retries, and dead-letter queues.

#### Evaluation
This hybrid option provides the optimal architecture for flash sale systems, balancing high throughput, low latency, data safety, and peak shaving.

## Consequences

### Positive Consequences
- Flash-sale purchase endpoints ingest massive traffic spikes asynchronously without crashing persistent databases.
- Read operations scale independently using Redis cache clusters.
- Zero over-selling is guaranteed through atomic memory operations and queued processing.
- WebSocket pushes notify users asynchronously once workers complete order persistence.
- Unpaid orders are automatically released back to stock using delay queues after 15 minutes.
- Audit logging is reliably triggered by events without blocking core purchasing logic.

### Negative Consequences
- System architecture is more complex than a standard monolithic application.
- Developers must manage eventual consistency and asynchronous messaging logic.
- Requires infrastructure monitoring for Kafka queue backlogs and Redis memory usage.

### Risk Mitigation
- Use Redis Lua scripts for atomic stock validation and deduction.
- Implement database unique composite indexes (`User_ID + Activity_ID`) to guarantee consumer idempotency.
- Implement dead-letter queues (DLQ) and retry mechanisms for failed order events.
- Deploy WebSocket fallback polling for client queue notification redundancy.

## Impact on Architecture
This decision determines the core container and component decomposition:
- Client UI Layer (Web/Mobile);
- API Gateway & Rate Limiting Container;
- CQRS Read Cache Container (Redis Cluster);
- Flash Sale Command Producer Container (Lua Script Execution);
- Message Broker Container (Apache Kafka);
- Asynchronous Worker Consumer Cluster;
- Persistence Database (MySQL Primary-Secondary);
- Unpaid Order Delay Queue Container;
- Notification Container (WebSocket);
- Audit Log Storage.

## Related Requirements
- **FR-1.3**: IP and User Rate Limiting
- **FR-2.1**: CQRS Multilevel Inventory Caching
- **FR-3.1**: Redis Lua Atomic Stock Pre-deduction
- **FR-3.2**: EDA Kafka Event Publication and Queueing
- **FR-3.5**: Idempotent Order Persistence and Zero Over-selling
- **FR-4.3**: Asynchronous Unpaid Order Stock Recovery
- **NFR-PER-1**: $100,000\text{ QPS}$ Read / $50,000\text{ TPS}$ Write Performance Targets
- **NFR-PER-2**: Sub-50ms Response Latency for Purchase Ingestion

## Related Architectural Characteristics
- Scalability
- Availability
- Performance & Low Latency
- Data Consistency & Safety
- Auditability
- Fault Tolerance & Peak Shaving
- Maintainability

## Final Decision
The selected architectural style is **Event-Driven Architecture (EDA) with CQRS Pattern**.
This architecture is appropriate for the Online Ticket Flash Sale System because it fulfills all key requirements of a high-concurrency ticketing platform: extreme peak shaving, low-latency ingestion, strict zero over-selling, isolated read/write scaling, and reliable background order persistence.
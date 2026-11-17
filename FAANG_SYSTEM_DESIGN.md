
# FAANG System Design — Interview Preparation

**Total: 47 exercises**
- 42 Core System Design Challenges
- 5 Architectural Evolution & Troubleshooting Challenges

**Target interview duration:** 45–60 minutes per exercise.

**Preparation objective:** Develop the ability to independently design scalable, reliable, maintainable distributed systems, explain architectural trade-offs, and reason about non-functional requirements.

---

# PART I — CORE SYSTEM DESIGN (42)

## 1. Distributed Systems & Infrastructure

### 1. Design a Rate Limiter (Cloudflare, Redis)

Design a distributed rate-limiting service that controls how frequently users, applications or API clients can access a system. Support configurable policies such as requests per second, per minute or per user, with different limits for different clients and endpoints. Consider algorithms such as Token Bucket, Leaky Bucket and Sliding Window, and determine how limits are enforced consistently across multiple application instances. The system should handle high request throughput, low decision latency and bursts of traffic. Consider availability during failures, distributed state management and the operational cost of maintaining counters.

### 2. Design an API Gateway (Kong, AWS API Gateway)

Design a central API entry point responsible for routing requests to multiple backend services. Support authentication, authorization, request validation, rate limiting, throttling, load balancing and API versioning. Consider how the gateway handles service discovery, backend failures, timeouts and retries. The system should support high throughput, low latency and horizontal scalability without becoming a single point of failure. Consider whether certain responsibilities belong in the gateway or in downstream services.

### 3. Design a URL Shortener (Bitly, TinyURL)

Design a service that converts long URLs into short, unique identifiers and redirects users to the original destinations. Support URL creation, redirection, optional expiration, custom aliases and basic analytics. Assume a large number of redirects compared with URL creation requests, potentially billions of stored mappings. Consider identifier generation, collision avoidance, caching, database partitioning and read/write traffic distribution. The redirection path should have very low latency and high availability.

### 4. Design a Distributed Cache (Redis, Memcached)

Design a distributed caching platform that allows multiple application instances to access shared cached data. Support key-value operations, expiration policies, eviction strategies and cache invalidation. Assume a high read/write workload with a working dataset potentially larger than the memory of a single machine. Consider data partitioning, replication, consistency, hot keys, cache stampedes and node failures. Minimise access latency while maintaining availability and predictable performance during scaling operations.

### 5. Design a Distributed Key-Value Store (DynamoDB, Cassandra)

Design a distributed database that provides key-based reads and writes across multiple nodes. Support data partitioning, replication, configurable consistency guarantees and horizontal scaling. Assume a potentially massive dataset with high read/write throughput and nodes distributed across multiple availability zones. Consider partition-key selection, replication strategies, quorum operations, conflict resolution and node recovery. Explain the trade-offs between consistency, availability, latency and durability.

### 6. Design a Distributed Message Queue (Kafka, RabbitMQ)

Design a messaging platform that enables producers to publish events and consumers to process them asynchronously. Support multiple topics or queues, message ordering, consumer groups, acknowledgements, retries and dead-letter handling. Assume high message throughput, potentially millions of messages per second across the platform, with varying message sizes and consumer processing speeds. Consider persistence, replication, partitioning, delivery guarantees and backpressure. Ensure the system can recover from broker failures without losing acknowledged messages.

### 7. Design a Distributed Locking Service (ZooKeeper, etcd)

Design a distributed coordination service that allows multiple application instances to acquire and release locks on shared resources. Support exclusive locking, lease expiration, automatic recovery and protection against abandoned locks. Assume clients and service nodes can fail independently, network partitions can occur, and locks must not remain permanently held. Consider lock ownership, lease renewal, fencing tokens, consistency guarantees and split-brain scenarios. The service should provide reliable coordination with predictable acquisition latency.

### 8. Design a Distributed Job Scheduler (Airflow, Kubernetes)

Design a platform that schedules and executes background jobs across a cluster of worker machines. Support one-time and recurring jobs, priorities, dependencies, retries, timeouts and execution history. Assume thousands or millions of scheduled jobs with varying execution durations and resource requirements. Consider job assignment, worker failures, duplicate execution, scheduling fairness and recovery after scheduler restarts. Support horizontal scaling and provide visibility into job status and execution failures.

### 9. Design a Workflow Orchestration Engine (Temporal, Camunda)

Design a platform that executes long-running business workflows consisting of multiple sequential or parallel activities. Support conditional branching, retries, timers, external events, compensation and human interaction. Assume workflows may run for hours, days or months, and individual activities may fail independently. Consider durable workflow state, event history, recovery, versioning and exactly-once business effects despite potentially repeated activity execution. The platform should support a large number of concurrent workflow instances while providing operational visibility.

### 10. Design a Feature Flag and Configuration Service (LaunchDarkly, Unleash)

Design a central platform for managing application configuration and dynamically enabling or disabling features across multiple services. Support targeted rollouts, user segmentation, percentage-based releases, environment-specific configuration and emergency kill switches. Assume thousands of application instances that need configuration updates with minimal latency. Consider configuration distribution, caching, consistency, propagation delays and behaviour during service outages. Applications should continue operating safely when the configuration platform becomes temporarily unavailable.

### 11. Design a Message Storage System at Scale (WhatsApp, Telegram)

Design a persistent message storage platform for a communication application that stores and retrieves messages between users and groups. Support chronological history, pagination, message delivery state, edits, deletions and retention policies. Assume hundreds of millions of users, substantial daily message volumes and potentially billions of stored messages. Consider data partitioning, conversation ordering, indexing, replication, hot conversations and efficient historical retrieval. The system should provide durable storage, low-latency reads and predictable performance as message history grows.

---

## 2. Search, Data Engineering & Discovery

### 12. Design a Search Engine (Google, Bing)

Design a web search platform that indexes documents from multiple sources and returns relevant results for user queries. Support crawling or ingestion, document indexing, ranking, filtering and query execution. Assume billions of documents and a very large number of concurrent searches. Consider inverted indexes, distributed indexing, ranking signals, query partitioning, freshness and index updates. Search results should be returned with low latency while supporting continuous ingestion and index maintenance.

### 13. Design a Web Crawler (Googlebot, Apache Nutch)

Design a distributed crawler that discovers and downloads web pages across the internet. Support URL discovery, deduplication, prioritisation, scheduling, robots.txt compliance and content extraction. Assume billions of URLs, a large number of crawler workers and websites with different response times and rate limits. Consider URL frontier management, politeness policies, retry handling, distributed coordination and crawl freshness. The system should scale horizontally while avoiding excessive load on external websites.

### 14. Design a Search Autocomplete System (Google, Elasticsearch)

Design a service that suggests search queries while users type into a search box. Support prefix-based matching, popularity ranking, personalisation and potentially regional or language-specific suggestions. Assume millions of users generating a large volume of requests, with extremely low response-time requirements. Consider indexing strategies, prefix matching, caching, ranking and frequent updates to suggestion popularity. The system should provide useful results even under high concurrency and support continuous updates to its underlying data.

### 15. Design an E-Commerce Product Catalogue and Search System (Amazon, Shopify)

Design a platform that manages product information and enables customers to search and filter a large product catalogue. Support product attributes, categories, variants, pricing, availability, faceted filtering and sorting. Assume millions of products, frequent catalogue updates and high read traffic during shopping peaks. Consider the relationship between the transactional product database and search indexes, eventual consistency, indexing pipelines and query performance. Product information should remain reliable while search results are sufficiently fresh.

### 16. Design a Real-Time Data Ingestion Pipeline (Apache Kafka, Apache Flink)

Design a distributed platform that ingests events from multiple producers and makes them available for downstream processing and analytics. Support high-volume streaming, event validation, schema evolution, buffering, partitioning and consumer subscriptions. Assume millions of events per second with variable payload sizes and unpredictable traffic spikes. Consider ordering, delivery guarantees, backpressure, late-arriving events, replay and fault recovery. The system should scale horizontally while maintaining predictable ingestion latency and data durability.

### 17. Design a Logging and Event Collection Platform (Splunk, Datadog)

Design a centralised platform that collects logs and operational events from thousands of applications and infrastructure components. Support ingestion, structured and unstructured events, indexing, filtering, querying, retention and archival. Assume very high event volumes, potentially terabytes of data per day, with significant ingestion bursts. Consider buffering, compression, partitioning, indexing cost, retention tiers and query latency. Ensure that temporary downstream failures do not cause widespread log loss.

### 18. Design a Metrics, Monitoring and Alerting Platform (Prometheus, Datadog)

Design a platform that collects, stores and queries time-series metrics from distributed applications and infrastructure. Support metric ingestion, aggregation, dashboards, alert rules and notification delivery. Assume millions of active time series with frequent samples and multiple users querying historical and recent data. Consider cardinality management, storage compression, downsampling, retention, query efficiency and alert evaluation latency. The monitoring platform itself should remain available and reliable even during infrastructure incidents.

### 19. Design an A/B Testing Platform (Optimizely, Statsig)

Design a platform that allows product teams to run controlled experiments by assigning users to different variants of a feature or experience. Support experiment configuration, audience targeting, consistent assignment, event collection and statistical reporting. Assume millions of users participating in multiple concurrent experiments, with events arriving asynchronously. Consider randomisation, sample-ratio mismatch, exposure tracking, experiment contamination and data consistency. Assignment should have low latency and remain stable for a given user throughout an experiment.

---

## 3. Social Networks, Communication & Collaboration

### 20. Design a News Feed System (Facebook, LinkedIn)

Design a personalised feed that aggregates content from users, pages or other entities a person follows. Support content publishing, feed generation, ranking, pagination and updates as new content appears. Assume hundreds of millions of users, with highly unequal follower counts and a read-heavy workload. Consider push versus pull feed generation, fan-out strategies, caching, ranking and handling extremely popular accounts. Feed retrieval should be fast while balancing freshness, relevance and infrastructure cost.

### 21. Design a Social Media Platform (Instagram, X)

Design a social networking platform where users can create profiles, publish posts, follow other users and interact through likes, comments and shares. Support media attachments, user relationships, privacy settings and content discovery. Assume hundreds of millions of users with high write volumes and significant traffic concentration around popular accounts or events. Consider storage, media delivery, feed integration, graph queries, consistency and abuse prevention. The platform should support continuous growth without degrading user-facing performance.

### 22. Design a Real-Time Chat System (WhatsApp, Slack)

Design a messaging platform that supports one-to-one and group conversations with real-time message delivery. Support online presence, typing indicators, message history, delivery/read receipts, attachments and offline message delivery. Assume millions of concurrent connections and potentially billions of messages exchanged daily. Consider persistent connections, message routing, ordering, storage, fan-out, reconnection and cross-device synchronisation. Ensure reliable message delivery while keeping latency low and supporting sudden connection spikes.

### 23. Design a Notification Service (Firebase, Amazon SNS)

Design a centralised service that delivers notifications to users through multiple channels, including push notifications, email and SMS. Support user preferences, templates, priorities, scheduling, retries and delivery tracking. Assume multiple applications generate millions of notification requests, with occasional traffic spikes. Consider asynchronous processing, provider integration, deduplication, rate limits, delivery guarantees and dead-letter handling. The service should isolate provider failures and prevent notification overload or duplicate delivery.

### 24. Design a Collaborative Document Editing System (Google Docs, Notion)

Design a multi-user document editing platform where several users can modify the same document concurrently. Support real-time updates, cursor presence, document history, permissions and offline editing with later synchronisation. Assume large numbers of concurrent editing sessions and potentially long-lived documents. Consider concurrent modification handling, conflict resolution using approaches such as Operational Transformation or CRDTs, persistence and recovery. The system should provide a responsive editing experience while preserving document correctness.

### 25. Design a Real-Time Collaborative Design Canvas (Figma, Miro)

Design a collaborative visual workspace where multiple users simultaneously manipulate objects on a shared canvas. Support object creation, movement, resizing, grouping, layers, comments and real-time presence. Assume large documents containing thousands of objects and multiple concurrent editors. Consider state synchronisation, concurrent operations, conflict resolution, efficient rendering updates, version history and reconnecting clients. The system should remain responsive during complex editing sessions without losing user changes.

---

## 4. Media, Streaming & Content Delivery

### 26. Design a Video Streaming Platform (Netflix, YouTube)

Design a platform that allows users to browse, search and stream pre-recorded video content. Support video uploads, transcoding into multiple resolutions and formats, content metadata, playback, recommendations and viewing history. Assume millions of concurrent viewers distributed geographically, with varying network bandwidth and device capabilities. Consider object storage, transcoding pipelines, CDN integration, adaptive bitrate streaming and content delivery costs. The system should provide reliable playback with minimal startup delay and buffering.

### 27. Design a Live Video Streaming Service (Twitch, YouTube Live)

Design a platform that enables creators to broadcast live video to large audiences in near real time. Support stream ingestion, transcoding, distribution, viewer subscriptions and optional live chat. Assume highly variable audience sizes, including streams watched by millions of simultaneous viewers. Consider stream replication, geographic distribution, adaptive bitrate, latency requirements, buffering and sudden traffic spikes. The system should balance playback quality, distribution cost and end-to-end latency.

---

## 5. Commerce, Payments & Financial Systems

### 28. Design an E-Commerce Platform (Amazon, Shopify)

Design an online shopping platform that supports product discovery, shopping carts, checkout, orders and order tracking. Integrate inventory, pricing, payment processing and fulfilment services. Assume millions of users, significant seasonal traffic spikes and a high read-to-write ratio for product browsing. Consider service boundaries, cart persistence, inventory consistency, checkout reliability and integration with external providers. The platform should prevent inconsistent orders and provide a reliable shopping experience even when individual downstream services fail.

### 29. Design a Payment Processing System (Stripe, Adyen)

Design a payment platform that accepts payment requests from merchants and coordinates their processing through external payment providers or financial institutions. Support authorisation, capture, refunds, payment status tracking, retries and reconciliation. Assume high transaction volumes, strict financial correctness requirements and unreliable external dependencies. Consider idempotency, duplicate requests, transaction state machines, asynchronous processing, auditability and failure recovery. Ensure that every transaction can be accurately accounted for, including cases where a provider times out after processing a payment.

### 30. Design a Digital Wallet and Account Balance Service (PayPal, Revolut)

Design a service that maintains user balances and processes deposits, withdrawals and transfers between accounts. Support concurrent transactions, balance checks, transaction history and account-level limits. Assume high transaction throughput and potentially multiple concurrent operations affecting the same account. Consider atomicity, isolation, ledger design, double-entry accounting, idempotency and consistency between available and posted balances. Financial correctness and auditability must be guaranteed even during failures and retries.

### 31. Design an Online Ticket Booking System (Ticketmaster, Eventbrite)

Design a platform where users can search for events, view available seats and purchase tickets. Support seat selection, temporary reservations, checkout, payment confirmation, cancellations and ticket issuance. Assume popular events may generate extremely concentrated traffic, with thousands of users competing for limited inventory simultaneously. Consider inventory consistency, reservation expiration, concurrent access, overselling prevention, payment failures and queue management. The system should remain fair and responsive during major ticket releases.

### 32. Design a Hotel and Flight Reservation System (Booking.com, Expedia)

Design a reservation platform that allows users to search accommodation or flights, check availability, compare prices and complete bookings. Support inventory management, temporary holds, price changes, cancellations and booking confirmation across external suppliers. Assume high search traffic, frequently changing inventory and concurrent booking attempts for limited resources. Consider availability synchronisation, stale search results, reservation holds, payment coordination and compensation when a multi-step booking fails. The system should avoid double bookings while providing a responsive search experience.

### 33. Design a Ride-Sharing Platform (Uber, Lyft)

Design a platform that connects passengers requesting rides with nearby drivers. Support driver location tracking, ride matching, fare estimation, trip lifecycle management, payments and cancellations. Assume millions of active users and drivers, with continuously changing geographic data and demand fluctuations. Consider geospatial indexing, real-time location updates, matching algorithms, concurrency, dispatch latency and regional scaling. The system should handle peak demand and preserve accurate ride state throughout the trip.

### 34. Design a Food Delivery Platform (DoorDash, Uber Eats)

Design a service that connects customers, restaurants and delivery couriers. Support restaurant discovery, menus, order placement, payment, courier assignment, order tracking and delivery confirmation. Assume significant demand fluctuations, geographical constraints and varying preparation and delivery times. Consider order state management, dispatch optimisation, location tracking, inventory/menu freshness and coordination between independent participants. The platform should provide reliable order progression and recover gracefully when restaurants or couriers become unavailable.

### 35. Design a Stock Price Alerting System (Yahoo Finance, TradingView)

Design a platform that monitors financial market prices and notifies users when specified conditions are met. Support price thresholds, percentage changes, recurring conditions, watchlists and multiple notification channels. Assume high-frequency market data ingestion, millions of user-defined alerts and potentially large bursts of triggered notifications. Consider event processing, rule evaluation, deduplication, latency, data accuracy and notification delivery. The system should evaluate alerts efficiently without missing relevant price movements.

---

## 6. Recommendations, Advertising & Personalisation

### 36. Design a Recommendation System (Netflix, Amazon)

Design a platform that recommends content or products to users based on their interests, behaviour and historical interactions. Support candidate generation, ranking, personalisation, contextual signals and feedback collection. Assume millions of users and a large catalogue of items, with recommendations generated under strict latency requirements. Consider offline model training, online inference, feature storage, cold-start problems, freshness and experimentation. The system should balance relevance, diversity, computational cost and recommendation quality.

### 37. Design an Online Advertising Platform (Google Ads, Meta Ads)

Design an advertising platform that connects advertisers with users and selects advertisements for available placements. Support campaign creation, targeting, bidding, budget management, ad selection, impression tracking and billing. Assume very high request volumes, potentially millions of ad-selection decisions per second, with extremely strict response-time requirements. Consider auction mechanisms, targeting indexes, budget consistency, frequency capping, fraud detection and event attribution. The platform should maximise advertising relevance and revenue while respecting campaign budgets and latency constraints.

### 38. Design a Music Recommendation Platform (Spotify, Apple Music)

Design a music discovery service that recommends tracks, albums and playlists based on listening behaviour and user preferences. Support personalised playlists, recommendations, collaborative filtering, contextual signals and continuous feedback from user interactions. Assume millions of users, a large music catalogue and frequent recommendation requests. Consider recommendation freshness, cold-start users, offline processing, online ranking, feature pipelines and low-latency delivery. The platform should balance personal relevance, content diversity and computational efficiency.

---

## 7. File Storage & Developer Platforms

### 39. Design a File Storage and Synchronisation Service (Dropbox, Google Drive)

Design a cloud storage platform that allows users to upload, download, organise and synchronise files across multiple devices. Support large files, metadata, folder structures, sharing permissions, version history and concurrent modifications. Assume millions of users and potentially petabytes of stored data. Consider chunked uploads, object storage, deduplication, synchronisation protocols, conflict resolution and CDN delivery. The system should provide durable storage and reliable synchronisation despite intermittent client connectivity.

### 40. Design an LLM Inference Serving Platform (OpenAI, Anthropic)

Design a platform that serves inference requests for large language models to multiple users and applications. Support synchronous and streaming responses, model selection, token-based quotas, authentication and request prioritisation. Assume high concurrency, variable prompt and output lengths, GPU memory constraints and potentially expensive inference operations. Consider request batching, token scheduling, model loading, GPU utilisation, KV-cache management, autoscaling and latency-versus-throughput trade-offs. The system should provide predictable performance while efficiently managing scarce computational resources.

### 41. Design a Distributed GPU Job Scheduling System (Kubernetes, Slurm)

Design a platform that schedules GPU-intensive workloads across a distributed cluster of machines. Support job submission, resource requirements, priorities, queues, placement constraints, preemption and execution monitoring. Assume heterogeneous GPU hardware, limited accelerator capacity and workloads ranging from short inference jobs to long-running model training. Consider resource allocation, fragmentation, fairness, GPU utilisation, fault recovery and multi-tenant isolation. The scheduler should maximise cluster utilisation while meeting workload priorities and service-level requirements.

### 42. Design a CI/CD Deployment Platform (GitHub Actions, Jenkins)

Design a platform that automates building, testing and deploying software across multiple environments. Support pipeline definitions, dependency management, distributed execution, build artefacts, secrets, approvals, rollback and execution history. Assume thousands of repositories and potentially large numbers of concurrent pipeline runs. Consider worker scheduling, build isolation, caching, resource quotas, security, pipeline reliability and deployment consistency. The platform should provide reproducible builds and safe deployments without becoming a bottleneck for development teams.

---

# PART II — ARCHITECTURAL EVOLUTION & TROUBLESHOOTING (5)

These exercises are deliberately different from the first 42: instead of designing a new system from scratch, the candidate is given an existing architecture, business constraints or operational problems and must evolve the design.

## 8. Architecture Evolution & Production Engineering

### 43. Migrate a Monolithic Application to Microservices (Amazon, Netflix)

An existing business-critical application is implemented as a monolith, with multiple functional domains sharing the same codebase and database. The organisation wants to introduce independently deployable services to improve scalability, maintainability and team autonomy. Design a migration strategy that allows the existing application to continue operating while functionality is gradually extracted. Consider domain boundaries, database decomposition, API contracts, data consistency, inter-service communication and deployment dependencies. Explain how to minimise migration risk, avoid a big-bang rewrite and maintain backward compatibility throughout the transition.

### 44. Design a Multi-Region Distributed Architecture (Netflix, Cloudflare)

An existing application operates in one geographical region and must expand to multiple regions to improve availability, reduce latency and support global traffic. Design an architecture that distributes application workloads and data across geographically separated regions. Consider regional traffic routing, data replication, consistency, failover, disaster recovery, split-brain scenarios and regional isolation. Define realistic recovery objectives (RTO/RPO) and explain how the system behaves during network partitions or complete regional outages. Balance availability, data correctness, latency and operational complexity.

### 45. Eliminate a Database Bottleneck in an Existing System (Instagram, Shopify)

An existing production application experiences increasing response times, database saturation and failed requests as traffic grows. You are given an architecture in which several application instances share a central relational database, and database utilisation has become the primary bottleneck. Investigate potential causes such as inefficient queries, locking, connection exhaustion, excessive writes and unbalanced access patterns. Propose an incremental strategy involving appropriate indexing, query optimisation, caching, read replicas, partitioning or workload separation where justified. Explain how to validate improvements using metrics and ensure that performance optimisations do not compromise data correctness.

### 46. Improve Reliability of a Distributed Transaction Processing Platform (Stripe, PayPal)

An existing distributed payment or order-processing platform occasionally experiences duplicate transactions, inconsistent states and partially completed operations when downstream services fail. Design an approach to improve transactional reliability across multiple independently deployed services. Consider idempotency, durable state transitions, transactional outbox, event delivery, retries, compensation, reconciliation and recovery procedures. Explain how the platform can distinguish successful, failed and uncertain external operations. Ensure that the proposed architecture preserves business invariants without relying on unrealistic assumptions about exactly-once delivery across distributed systems.

### 47. Design a Zero-Downtime Database Migration Strategy (GitHub, Shopify)

An existing production application must migrate a large database to a new schema, storage engine or database platform without interrupting customer-facing operations. Design a migration approach that supports continuous reads and writes while data is progressively transformed and transferred. Consider schema compatibility, backfills, change-data capture, dual writes or dual reads, consistency validation and rollback strategies. Explain how to handle concurrent updates, failed migration batches and differences between the old and new data models. The migration should minimise operational risk and provide a controlled path to decommissioning the old database.

---

# APPENDIX — INTERVIEW EXECUTION CHECKLIST

For every exercise, aim to follow a consistent 45–60 minute interview structure.

## Suggested Time Allocation

| Phase | Time | Expected outcome |
|---|---:|---|
| 1. Requirements clarification | 5–7 min | Functional requirements, users, use cases and scope |
| 2. Non-functional requirements | 5 min | Scale, latency, availability, consistency and durability |
| 3. Back-of-the-envelope estimation | 5–7 min | Traffic, storage, bandwidth and capacity estimates |
| 4. High-level architecture | 10–12 min | Core components, APIs, data flow and communication |
| 5. Deep dive | 12–15 min | Critical components, algorithms, data models and bottlenecks |
| 6. Trade-offs and failure scenarios | 5–7 min | Scalability, resilience, consistency and alternatives |
| 7. Wrap-up | 2 min | Recap, limitations and possible future improvements |

## Standard Non-Functional Requirements Checklist

For each problem, explicitly consider:

- **Scalability:** Expected users, requests per second, data volume and growth.
- **Availability:** Target uptime, failure tolerance and acceptable degradation.
- **Latency:** Response-time requirements, especially p95 and p99.
- **Consistency:** Strong consistency, eventual consistency or other required guarantees.
- **Durability:** Data persistence, replication, backup and recovery.
- **Reliability:** Retries, idempotency, fault isolation and graceful degradation.
- **Security:** Authentication, authorisation, encryption and tenant isolation.
- **Observability:** Logging, metrics, distributed tracing and alerting.
- **Cost efficiency:** Compute, storage, bandwidth and operational overhead.
- **Maintainability:** Deployment, versioning, extensibility and operational complexity.

## Standard Back-of-the-Envelope Estimation Checklist

Depending on the problem, estimate:

- Daily Active Users (DAU) and Monthly Active Users (MAU).
- Average and peak Requests Per Second (RPS).
- Read-to-write ratio.
- Average request and response payload sizes.
- Data generated per day, month and year.
- Total storage requirements and retention period.
- Network bandwidth and throughput.
- Number of application instances, workers or partitions.
- Cache memory requirements and expected hit ratio.
- Replication overhead and failure capacity.

**Important:** Treat all numerical values as interview assumptions to be clarified with the interviewer, not as fixed requirements. State your assumptions explicitly and use order-of-magnitude estimates rather than false precision.

## Final Preparation Principle

Do not memorise 47 architecture diagrams.

For each exercise, practise:
1. Identifying the core problem and its constraints.
2. Establishing the most important functional and non-functional requirements.
3. Deriving a reasonable architecture from those requirements.
4. Identifying the most challenging component and exploring it in depth.
5. Explaining architectural trade-offs and alternative approaches.
6. Reasoning about failures, scaling limits and operational consequences.

**The objective is architectural reasoning, not architectural memorisation.**

# AWS Service Selection & Alternatives

## 1. API Gateway & Edge Services

### Selected Services

| Service | Purpose | Key Benefits |
|---------|---------|--------------|
| **Amazon API Gateway** | REST & WebSocket APIs | Managed throttling (10,000 RPS), built-in caching, request validation, AWS WAF integration |
| **CloudFront** | CDN & static asset delivery | 450+ global edge locations, DDoS protection via AWS Shield, SSL/TLS termination |
| **Route53** | DNS with health checks | Latency-based routing, health-based failover, 100% SLA |
| **AWS WAF** | Web Application Firewall | Managed rules for OWASP Top 10, rate limiting, geo-blocking |

### Alternatives Considered

**Application Load Balancer (ALB)**
- Pros: Lower cost per request, supports HTTP/2
- Cons: No built-in throttling, requires manual WebSocket management
- Decision: API Gateway chosen for managed WebSocket and throttling

**NGINX on EC2**
- Pros: Full control, highly customizable
- Cons: Self-managed, scaling complexity, no built-in WAF
- **Decision**: API Gateway chosen to reduce operational overhead

**Cloudflare**
- Pros: Excellent DDoS protection, global performance
- Cons: Multi-cloud complexity, AWS constraint violation
- **Decision**: CloudFront + WAF meets requirements within AWS

---

## 2. Compute Layer (Trading Engine)

### Selected Services

| Service | Purpose | Key Benefits |
|---------|---------|--------------|
| **Amazon ECS on Fargate** | Containerized matching engine | Serverless compute, auto-scaling, no server management, sub-second scaling |
| **AWS Lambda** | Event-driven processors | Auto-scaling, pay-per-execution, built-in retries, 15-min timeout |
| **Amazon SQS FIFO** | Order queue | Exactly-once delivery, message deduplication, 3,000 TPS per queue |

### Alternatives Considered

**Amazon EKS (Kubernetes)**
- Pros: Advanced orchestration, portability, complex deployments
- Cons: Overhead for 500 RPS, higher cost, steeper learning curve
- Decision: ECS Fargate chosen for simpler operations at this scale

**EC2 Auto Scaling Group**
- Pros: Full control, potentially lower cost
- Cons: Manual scaling delays (minutes), patching overhead, instance management
- Decision: Fargate chosen for faster scaling and reduced ops

**Apache Kafka (MSK)**
- Pros: Higher throughput, advanced streaming features
- Cons: Overkill for 500 RPS, higher operational complexity and cost
- Decision: SQS FIFO chosen for simplicity and exactly-once semantics

**RabbitMQ on EC2**
- Pros: Feature-rich, flexible routing
- Cons: Self-managed, HA complexity, no managed service
- **Decision**: SQS FIFO chosen for managed service benefits

---

## 3. Data Storage Layer

### Selected Services

| Service | Purpose | Key Benefits |
|---------|---------|--------------|
| **Amazon Aurora PostgreSQL** | Primary transactional DB | 5x MySQL performance, automated backups, 15 read replicas, Multi-AZ HA |
| **Amazon DynamoDB** | User sessions & hot data | Single-digit ms latency, auto-scaling, 99.999% SLA, global tables |
| **Amazon ElastiCache (Redis)** | Orderbook cache | Sub-millisecond latency, cluster mode, automatic failover, backup/restore |
| **Amazon S3** | Historical data & backups | 99.999999999% durability, lifecycle policies, cross-region replication |
| **Amazon Kinesis Data Streams** | Real-time event streaming | 200ms latency, 1MB records, ordered delivery, replay capability |

### Alternatives Considered

**RDS MySQL**
- Pros: Familiar, stable, good performance
- Cons: Lower performance than Aurora, fewer read replicas (5 vs 15)
- Decision: Aurora chosen for better HA and performance at scale

**Self-managed PostgreSQL on EC2**
- Pros: Full control, custom extensions
- Cons: Manual backups, replication complexity, patching overhead
- **Decision**: Aurora chosen for automated operations

**MongoDB Atlas / DocumentDB**
- Pros: Flexible schema, horizontal scaling
- Cons: Eventual consistency challenges for trading, higher cost
- **Decision**: Aurora + DynamoDB hybrid chosen for strong consistency where needed

**Redis on EC2**
- Pros: Lower cost, full Redis config control
- Cons: Self-managed HA, manual backups, failover complexity
- **Decision**: ElastiCache chosen for managed HA and automatic failover

**Apache Cassandra / Keyspaces**
- Pros: Linear scalability, multi-region writes
- Cons: Eventual consistency unsuitable for trading, operational complexity
- **Decision**: DynamoDB chosen for simpler operations and strong consistency

**Amazon Timestream**
- Pros: Purpose-built for time-series data, automated tiering
- Cons: Limited querying compared to S3 + Athena, newer service
- **Decision**: S3 + Athena chosen for flexibility and cost

---

## 4. Real-time Communications

### Selected Services

| Service | Purpose | Key Benefits |
|---------|---------|--------------|
| **API Gateway WebSocket** | Bidirectional real-time updates | Managed connections, auto-scaling, pay-per-connection, integration with Lambda |
| **Amazon EventBridge** | Event routing & fan-out | Content-based filtering, archive/replay, 300+ targets |

### Alternatives Considered

**AWS AppSync (GraphQL Subscriptions)**
- Pros: GraphQL subscriptions, built-in caching, offline sync
- Cons: GraphQL overhead for simple pub/sub, REST API already chosen
- **Decision**: API Gateway WebSocket chosen for consistency with REST API

**Self-managed Socket.io on EC2/ECS**
- Pros: Full control, rich client libraries
- Cons: Connection state management, scaling complexity, no managed service
- **Decision**: API Gateway chosen for serverless scaling

**Amazon SNS + SQS Fan-out**
- Pros: Simple pub/sub, reliable delivery
- Cons: No direct client connections, requires polling
- **Decision**: WebSocket chosen for true real-time push

**Pusher / Ably (3rd party)**
- Pros: Easy integration, managed service
- Cons: AWS constraint violation, data privacy concerns
- **Decision**: AWS-native chosen for compliance

---

## 5. Authentication & Security

### Selected Services

| Service | Purpose | Key Benefits |
|---------|---------|--------------|
| **Amazon Cognito** | User authentication & JWT | Built-in user pools, OAuth 2.0, MFA, integration with API Gateway |
| **AWS Secrets Manager** | Credential management | Automatic rotation, encryption, fine-grained IAM policies |
| **AWS KMS** | Encryption key management | FIPS 140-2 validated, automatic key rotation, audit logging |
| **AWS Certificate Manager** | SSL/TLS certificates | Free certificates, automatic renewal, integrated with CloudFront/ALB |

### Alternatives Considered

**Auth0**
- Pros: Rich features, extensive integrations, good UX
- Cons: Additional cost, external dependency, data residency
- **Decision**: Cognito chosen for AWS integration and cost

**Keycloak on EC2**
- Pros: Open source, highly customizable
- Cons: Self-managed, HA complexity, operational overhead
- **Decision**: Cognito chosen for managed service

**HashiCorp Vault**
- Pros: Advanced features, multi-cloud support
- Cons: Self-managed, complexity, cost
- **Decision**: Secrets Manager chosen for simpler operations

**Let's Encrypt**
- Pros: Free, widely supported
- Cons: Manual renewal process, no CloudFront integration
- **Decision**: ACM chosen for automatic renewal and AWS integration

---

## 6. Observability & Monitoring

### Selected Services

| Service | Purpose | Key Benefits |
|---------|---------|--------------|
| **Amazon CloudWatch** | Metrics, logs, alarms | Native AWS integration, 1-second metrics, anomaly detection |
| **AWS X-Ray** | Distributed tracing | End-to-end request tracking, service maps, latency analysis |
| **CloudWatch Logs Insights** | Log analytics | SQL-like query language, fast search, built-in dashboards |

### Alternatives Considered

**Datadog**
- Pros: Superior UI, advanced analytics, ML-based anomaly detection
- Cons: $15-50/host/month, external dependency, data transfer costs
- **Decision**: CloudWatch chosen for cost and native integration

**New Relic**
- Pros: Excellent APM, transaction tracing
- Cons: High cost at scale, external dependency
- **Decision**: CloudWatch + X-Ray chosen for cost

**Prometheus + Grafana on EKS**
- Pros: Open source, highly customizable, rich ecosystem
- Cons: Self-managed, storage challenges, operational overhead
- **Decision**: CloudWatch chosen for managed service

**ELK Stack (Elasticsearch + Kibana)**
- Pros: Powerful search, visualization flexibility
- Cons: High operational overhead, expensive at scale, cluster management
- **Decision**: CloudWatch Logs Insights chosen for simpler operations

---

## Cost Comparison Summary

| Service Category | Selected | Monthly Cost (Est.) | Alternative | Monthly Cost (Est.) |
|-----------------|----------|--------------------:|-------------|--------------------:|
| API Gateway | API Gateway | $1,050 (500 RPS) | ALB | $450 |
| Compute | ECS Fargate | $730 (10 tasks) | EC2 (t3.medium) | $304 |
| Database | Aurora + DynamoDB | $1,200 | RDS MySQL | $890 |
| Cache | ElastiCache Redis | $400 | Redis on EC2 | $180 |
| Monitoring | CloudWatch | $200 | Datadog | $1,500 |
| **TOTAL** | | **$3,580/month** | | **$3,324/month** |

**Decision**: Selected architecture prioritizes **operational simplicity** and **high availability** over pure cost optimization.
- Reduced operational overhead (no server patching, HA management)
- Faster time to market
- Built-in scaling and resilience
- Better SLAs (99.95%+ across services)
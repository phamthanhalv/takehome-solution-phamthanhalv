---

## System Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         CLIENT LAYER                                    │
│                                                                         │
│    ┌──────────────┐              ┌──────────────┐                       │
│    │ Web Browser  │              │  Mobile App  │                       │
│    └──────┬───────┘              └──────┬───────┘                       │
└───────────┼──────────────────────────────┼──────────────────────────────┘
            │                              │
            └──────────────┬───────────────┘
                           │
┌──────────────────────────▼────────────────────────────────────────────────┐
│                    AWS GLOBAL - EDGE & CDN                                │
│                                                                           │
│  ┌──────────────┐     ┌─────────────┐     ┌──────────────┐                │
│  │  Route53 DNS │────▶│  CloudFront │────▶│   AWS WAF    │                │
│  │              │     │     CDN     │     │   Firewall   │                │
│  └──────────────┘     └─────────────┘     └──────┬───────┘                │
└─────────────────────────────────────────────────────┼─────────────────────┘
                                                     │
┌─────────────────────────────────────────────────────▼─────────────────────┐
│                    AWS REGION - us-east-1                                 │
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────┐          │
│  │              API LAYER (Multi-AZ)                           │          │
│  │                                                             │          │
│  │  ┌────────────────┐        ┌─────────────────┐              │          │
│  │  │  API Gateway   │        │  API Gateway    │              │          │
│  │  │  REST API      │        │  WebSocket API  │              │          │
│  │  └───────┬────────┘        └────────┬────────┘              │          │
│  │          │                          │                       │          │
│  │          └──────────┬───────────────┘                       │          │
│  │                     │                                       │          │
│  │          ┌──────────▼──────────┐                            │          │
│  │          │  Amazon Cognito     │                            │          │
│  │          │  Authentication     │                            │          │
│  │          └─────────────────────┘                            │          │
│  └──────────────────────┬──────────────────────────────────────┘          │
│                         │                                                 │
│  ┌──────────────────────▼──────────────────────────────────────┐          │
│  │        APPLICATION LAYER (Multi-AZ: 1a, 1b, 1c)             │          │
│  │                                                             │          │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │          │
│  │  │ ECS Fargate  │  │ ECS Fargate  │  │     SQS      │       │          │
│  │  │  Trading     │  │  Trading     │  │  FIFO Queue  │       │          │
│  │  │  Engine      │  │  Engine      │  │   Orders     │       │          │
│  │  │   (AZ-1a)    │  │   (AZ-1b)    │  │              │       │          │
│  │  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘       │          │
│  │         │                  │                 │              │          │
│  │         └──────────┬───────┴─────────────────┘              │          │
│  │                    │                                        │          │
│  │         ┌──────────▼──────────┐                             │          │
│  │         │   AWS Lambda        │                             │          │
│  │         │   Order Processing  │                             │          │
│  │         └─────────────────────┘                             │          │
│  └──────────────────────┬──────────────────────────────────────┘          │
│                         │                                                 │
│  ┌──────────────────────▼──────────────────────────────────────┐          │
│  │              CACHE LAYER (Multi-AZ)                         │          │
│  │                                                             │          │
│  │  ┌──────────────────┐         ┌──────────────────┐          │          │
│  │  │ ElastiCache      │         │ ElastiCache      │          │          │
│  │  │ Redis Primary    │────────▶│ Redis Replica    │          │          │
│  │  │    (AZ-1a)       │  Async  │    (AZ-1b)       │          │          │
│  │  └──────────────────┘         └──────────────────┘          │          │
│  └──────────────────────┬──────────────────────────────────────┘          │
│                         │                                                 │
│  ┌──────────────────────▼──────────────────────────────────────┐          │
│  │           DATA LAYER (Multi-AZ: 1a, 1b, 1c)                 │          │
│  │                                                             │          │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │          │
│  │  │   Aurora     │  │   Aurora     │  │   Aurora     │       │          │
│  │  │  PostgreSQL  │─▶│  PostgreSQL  │  │  PostgreSQL  │       │          │
│  │  │   Writer     │  │   Reader 1   │  │   Reader 2   │       │          │
│  │  │   (AZ-1a)    │  │   (AZ-1b)    │  │   (AZ-1c)    │       │          │
│  │  └──────────────┘  └──────────────┘  └──────────────┘       │          │
│  │                                                             │          │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │          │
│  │  │  DynamoDB    │  │   Kinesis    │  │      S3      │       │          │
│  │  │ User Sessions│  │ Data Streams │  │  Historical  │       │          │
│  │  │  & Wallets   │  │Trade Events  │  │     Data     │       │          │
│  │  └──────────────┘  └──────────────┘  └──────────────┘       │          │
│  └─────────────────────────────────────────────────────────────┘          │
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────┐          │
│  │        MONITORING & OBSERVABILITY                           │          │
│  │                                                             │          │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │          │
│  │  │  CloudWatch  │  │    X-Ray     │  │  CloudTrail  │       │          │
│  │  │Metrics & Logs│  │   Tracing    │  │  Audit Logs  │       │          │
│  │  └──────────────┘  └──────────────┘  └──────────────┘       │          │
│  └─────────────────────────────────────────────────────────────┘          │
└───────────────────────────────────────────────────────────────────────────┘
```

---

## Request Flow: Order Placement

```
┌────────┐                                                           
│ Client │                                                           
└───┬────┘                                                           
    │                                                                
    │ 1. POST /api/v1/order (HTTPS)                                
    ▼                                                                
┌─────────────────┐                                                 
│  API Gateway    │ 2. Rate limiting check (1ms)                   
│                 │    ✓ Under 10,000 RPS limit                    
└───┬─────────────┘                                                 
    │                                                                
    │ 3. Validate JWT token                                        
    ▼                                                                
┌─────────────────┐                                                 
│  AWS Cognito    │ 4. Token validation (2ms)                      
│                 │    ✓ Valid user session                        
└───┬─────────────┘                                                 
    │                                                                
    │ 5. Forward to Trading Engine                                 
    ▼                                                                
┌─────────────────┐                                                 
│  ECS Fargate    │ 6. Order validation (5ms)                      
│ Trading Engine  │    - Check order format                        
│                 │    - Validate symbol/price                     
└───┬─────────────┘                                                 
    │                                                                
    │ 7. Check user balance                                        
    ▼                                                                
┌─────────────────┐                                                 
│   DynamoDB      │ 8. Get wallet balance (3ms)                    
│                 │    ✓ Sufficient funds                          
└───┬─────────────┘                                                 
    │                                                                
    │ 9. Get current orderbook                                     
    ▼                                                                
┌─────────────────┐                                                 
│ Redis Cache     │ 10. Fetch orderbook (2ms)                      
│                 │     Cache HIT                                  
└───┬─────────────┘                                                 
    │                                                                
    │ 11. Match order against orderbook                            
    ▼                                                                
┌─────────────────┐                                                 
│  ECS Fargate    │ 12. In-memory matching (10ms)                  
│ Trading Engine  │     Found partial match                        
└───┬─────────────┘                                                 
    │                                                                
    │ 13. Enqueue for persistence                                  
    ▼                                                                
┌─────────────────┐                                                 
│   SQS FIFO      │ 14. Enqueue order (5ms)                        
│   Queue         │     Exactly-once delivery                      
└───┬─────────────┘                                                 
    │                                                                
    │ 15. Update orderbook cache                                   
    ▼                                                                
┌─────────────────┐                                                 
│ Redis Cache     │ 16. Update cache (2ms)                         
│                 │     SET orderbook:BTCUSDT                      
└─────────────────┘                                                 
    │                                                                
    │ 17. Return order accepted response                           
    ▼                                                                
┌────────┐                                                           
│ Client │ 18. HTTP 202 Accepted (Total: ~30ms)                   
└────────┘     {"order_id": "12345", "status": "accepted"}         
                                                                     
                                                                     
    ╔════════════════════════════════════════════════════════╗      
    ║         ASYNC PROCESSING (After response)              ║      
    ╚════════════════════════════════════════════════════════╝      
                                                                     
┌─────────────────┐                                                 
│   SQS FIFO      │ 19. Trigger Lambda (event-driven)             
└───┬─────────────┘                                                 
    │                                                                
    ▼                                                                
┌─────────────────┐                                                 
│  AWS Lambda     │ 20. Process order (async)                      
│Order Processor  │     - Persist to Aurora                        
└───┬─────────────┘     - Update wallet                           
    │                   - Publish trade event                      
    ├─────────────────────────────────┐                            
    │                                 │                            
    ▼                                 ▼                            
┌─────────────────┐           ┌─────────────────┐                  
│ Aurora DB       │           │   DynamoDB      │                  
│                 │           │                 │                  
│ INSERT orders   │           │ UPDATE wallet   │                  
│ INSERT trades   │           │ balance         │                  
└─────────────────┘           └─────────────────┘                  
    │                                                                
    ▼                                                                
┌─────────────────┐                                                 
│ Kinesis Stream  │ 21. Publish trade event                        
│                 │     For analytics & audit                      
└───┬─────────────┘                                                 
    │                                                                
    ▼                                                                
┌─────────────────┐                                                 
│   S3 Bucket     │ 22. Archive to S3 (eventual)                   
│ Historical Data │     For compliance & backups                   
└─────────────────┘                                                 
```

**Latency Breakdown:**
- API Gateway: 1ms (throttling check)
- Cognito: 2ms (JWT validation)
- ECS validation: 5ms (order checks)
- DynamoDB read: 3ms (wallet balance)
- Redis read: 2ms (orderbook fetch)
- Order matching: 10ms (in-memory)
- SQS enqueue: 5ms (queue write)
- Redis update: 2ms (cache update)
- **Total: ~30ms** 

---

## Real-time Market Data Flow

**WebSocket Push Architecture**

```
┌────────┐                                                           
│ Client │ 1. Establish WebSocket connection                       
└───┬────┘                                                           
    │                                                                
    │ wss://api.example.com/ws                                     
    ▼                                                                
┌─────────────────────────────────────┐                            
│     API Gateway WebSocket           │                            
│                                     │                            
│  ┌────────────────────────────┐     │                            
│  │ Connection Table           │     │                            
│  │                            │     │                            
│  │ ConnectionId: abc123       │     │                            
│  │ UserId: user_456           │     │                            
│  │ Subscriptions: [BTCUSDT]   │     │                            
│  └────────────────────────────┘     │                            
└───┬─────────────────────────────────┘                            
    │                                                                
    │ 2. Subscribe to symbol                                       
    │ {"action": "subscribe", "symbol": "BTCUSDT"}                 
    ▼                                                                
┌─────────────────┐                                                 
│  ECS Fargate    │ 3. Add to subscription set                     
│ Trading Engine  │                                                 
└───┬─────────────┘                                                 
    │                                                                
    ▼                                                                
┌─────────────────┐                                                 
│ Redis Cache     │ 4. SADD subscribers:BTCUSDT abc123            
│                 │    Store connection mapping                    
└─────────────────┘                                                 
                                                                     
         ... Trade execution happens ...                            
                                                                     
┌─────────────────┐                                                 
│  ECS Fargate    │ 5. Order matched! New trade                    
│ Trading Engine  │    Price: $50,000                              
└───┬────┬────┬───┘                                                 
    │    │    │                                                     
    │    │    │ 6. Fan-out to multiple destinations                
    │    │    │                                                     
    │    │    ▼                                                     
    │    │ ┌─────────────────┐                                     
    │    │ │ Redis Cache     │ 7. Update price cache               
    │    │ │                 │    SET price:BTCUSDT 50000          
    │    │ └─────────────────┘                                     
    │    │                                                           
    │    ▼                                                           
    │ ┌─────────────────┐                                           
    │ │ Kinesis Stream  │ 8. Publish trade event                   
    │ │                 │    For analytics pipeline                
    │ └─────────────────┘                                           
    │                                                                
    │ 9. Get all subscribers                                       
    ▼                                                                
┌─────────────────┐                                                 
│ Redis Cache     │ 10. SMEMBERS subscribers:BTCUSDT              
│                 │     Returns: [abc123, def456, ghi789]         
└───┬─────────────┘                                                 
    │                                                                
    │ 11. Push to all WebSocket connections                        
    ▼                                                                
┌─────────────────────────────────────┐                            
│     API Gateway WebSocket           │                            
│                                     │                            
│  POST /@connections/abc123          │                            
│  POST /@connections/def456          │                            
│  POST /@connections/ghi789          │                            
└───┬─────────────────────────────────┘                            
    │                                                                
    │ 12. WebSocket message                                        
    ▼                                                                
┌────────┐                                                           
│ Client │ {"event": "trade",                                      
└────────┘  "symbol": "BTCUSDT",                                   
            "price": 50000,                                         
            "quantity": 0.5,                                        
            "timestamp": "2026-02-17T10:00:00Z"}                   
```
---

## Network Architecture

### VPC Layout (10.0.0.0/16)

```
┌─────────────────────────────────────────────────────────────────────┐
│                    INTERNET GATEWAY (IGW)                           │
└──────────────┬──────────────────────────────────┬───────────────────┘
               │                                   │
               │                                   │
┌──────────────▼──────────────┐   ┌───────────────▼──────────────┐
│  PUBLIC SUBNET (AZ-1a)      │   │  PUBLIC SUBNET (AZ-1b)       │
│  10.0.1.0/24                │   │  10.0.2.0/24                 │
│                             │   │                              │
│  ┌───────────────────────┐  │   │  ┌───────────────────────┐   │
│  │   NAT Gateway         │  │   │  │   NAT Gateway         │   │
│  │   Elastic IP          │  │   │  │   Elastic IP          │   │
│  └───────────────────────┘  │   │  └───────────────────────┘   │
│                             │   │                              │
│  ┌───────────────────────┐  │   │  ┌───────────────────────┐   │
│  │ Application LB        │  │   │  │ Application LB        │   │
│  │ (for ECS if needed)   │  │   │  │ (for ECS if needed)   │   │
│  └───────────────────────┘  │   │  └───────────────────────┘   │
└──────────────┬──────────────┘   └───────────────┬──────────────┘
               │                                   │
               │                                   │
┌──────────────▼──────────────┐   ┌───────────────▼──────────────┐
│ PRIVATE APP SUBNET (AZ-1a)  │   │ PRIVATE APP SUBNET (AZ-1b)   │
│ 10.0.10.0/24                │   │ 10.0.11.0/24                 │
│                             │   │                              │
│  ┌───────────────────────┐  │   │  ┌───────────────────────┐   │
│  │   ECS Tasks           │  │   │  │   ECS Tasks           │   │
│  │   Trading Engine      │  │   │  │   Trading Engine      │   │
│  │   (10 tasks)          │  │   │  │   (10 tasks)          │   │
│  └───────────────────────┘  │   │  └───────────────────────┘   │
│                             │   │                              │
│  ┌───────────────────────┐  │   │  ┌───────────────────────┐   │
│  │   Lambda Functions    │  │   │  │   Lambda Functions    │   │
│  │   (in VPC)            │  │   │  │   (in VPC)            │   │
│  └───────────────────────┘  │   │  └───────────────────────┘   │
└──────────────┬──────────────┘   └───────────────┬──────────────┘
               │                                   │
               │                                   │
┌──────────────▼──────────────┐   ┌───────────────▼──────────────┐
│ PRIVATE DATA SUBNET (AZ-1a) │   │ PRIVATE DATA SUBNET (AZ-1b)  │
│ 10.0.20.0/24                │   │ 10.0.21.0/24                 │
│                             │   │                              │
│  ┌───────────────────────┐  │   │  ┌───────────────────────┐   │
│  │ Aurora Writer         │  │   │  │ Aurora Reader         │   │
│  │ PostgreSQL            │  │   │  │ PostgreSQL            │   │
│  └───────────────────────┘  │   │  └───────────────────────┘   │
│                             │   │                              │
│  ┌───────────────────────┐  │   │  ┌───────────────────────┐   │
│  │ ElastiCache Primary   │  │   │  │ ElastiCache Replica   │   │
│  │ Redis                 │  │   │  │ Redis                 │   │
│  └───────────────────────┘  │   │  └───────────────────────┘   │
└─────────────────────────────┘   └──────────────────────────────┘

               ┌───────────────────────────────┐
               │ PRIVATE DATA SUBNET (AZ-1c)   │
               │ 10.0.22.0/24                  │
               │                               │
               │  ┌───────────────────────┐    │
               │  │ Aurora Reader         │    │
               │  │ PostgreSQL            │    │
               │  └───────────────────────┘    │
               └───────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                  VPC ENDPOINTS (PrivateLink)                        │
│                                                                     │
│  • S3 Gateway Endpoint (no data transfer cost)                      │
│  • DynamoDB Gateway Endpoint (no data transfer cost)                │
│  • Secrets Manager Interface Endpoint                               │
│  • CloudWatch Logs Interface Endpoint                               │
│  • ECR Interface Endpoint (Docker image pulls)                      │
└─────────────────────────────────────────────────────────────────────┘
```

**Security Groups:**

| Resource | Inbound | Outbound |
|----------|---------|----------|
| **ALB** | 443 from Internet | 8080 to ECS |
| **ECS Tasks** | 8080 from ALB | 5432 to Aurora, 6379 to Redis, 443 to DynamoDB |
| **Aurora** | 5432 from ECS/Lambda | None (managed) |
| **ElastiCache** | 6379 from ECS/Lambda | None (managed) |

---

## High Availability Architecture

### Multi-AZ Deployment

```
╔═══════════════════════════════════════════════════════════════╗
║                       REGION: us-east-1                       ║
╚═══════════════════════════════════════════════════════════════╝

┌────────────────────┐  ┌────────────────────┐  ┌────────────────────┐
│  AVAILABILITY      │  │  AVAILABILITY      │  │  AVAILABILITY      │
│   ZONE 1a          │  │   ZONE 1b          │  │   ZONE 1c          │
├────────────────────┤  ├────────────────────┤  ├────────────────────┤
│                    │  │                    │  │                    │
│ ┌────────────────┐ │  │ ┌────────────────┐ │  │                    │
│ │  ECS Tasks     │ │  │ │  ECS Tasks     │ │  │                    │
│ │  (5 instances) │ │  │ │  (5 instances) │ │  │                    │
│ └────────────────┘ │  │ └────────────────┘ │  │                    │
│                    │  │                    │  │                    │
│ ┌────────────────┐ │  │ ┌────────────────┐ │  │                    │
│ │ Redis PRIMARY  │◀┼──┼▶│ Redis REPLICA  │ │  │                    │
│ │                │ │  │ │                │ │  │                    │
│ │ Auto-failover  │ │  │ │ Sync replica   │ │  │                    │
│ └────────────────┘ │  │ └────────────────┘ │  │                    │
│                    │  │                    │  │                    │
│ ┌────────────────┐ │  │ ┌────────────────┐ │  │ ┌────────────────┐ │
│ │ Aurora WRITER  │─┼──┼▶│ Aurora READER  │ │  │ │ Aurora READER  │ │
│ │                │ │  │ │                │ │  │ │                │ │
│ │ Primary node   │ │  │ │ Slave node     │ │  │ │ Slave node     │ │
│ └────────────────┘ │  │ └────────────────┘ │  │ └────────────────┘ │
│                    │  │                    │  │                    │
└────────────────────┘  └────────────────────┘  └────────────────────┘
```

### Failure Scenarios

**Scenario 1: Single AZ Failure (AZ-1a goes down)**

```
Before:                          After (< 1 minute):
┌──────┐  ┌──────┐               ┌──────┐  ┌──────┐
│ AZ-1a│  │ AZ-1b│               │ AZ-1a│  │ AZ-1b│
├──────┤  ├──────┤               ├──────┤  ├──────┤
│ 5 ECS│  │ 5 ECS│               │  ✗   │  │10 ECS│ ← Auto-scaled
│Redis1│  │Redis2│               │  ✗   │  │Redis1│ ← Promoted
│Aurora│  │Aurora│               │  ✗   │  │Aurora│ ← Promoted
│Writer│  │Reader│               │      │  │Writer│
└──────┘  └──────┘               └──────┘  └──────┘

Traffic: 100% across both AZs    Traffic: 100% to AZ-1b
Impact: ZERO - automatic failover
```

---

## Auto-Scaling Configuration

### ECS Service Auto-Scaling

```
┌─────────────────────────────────────────────────────────────┐
│              CloudWatch Metrics                             │
│                                                             │
│  CPU Utilization > 70% for 5 minutes                        │
│  Memory Utilization > 80% for 5 minutes                     │
│  Request Count > 50 per task for 2 minutes                  │
└──────────────────┬──────────────────────────────────────────┘
                   │
                   │ Trigger Scale-Out Policy
                   ▼
┌─────────────────────────────────────────────────────────────┐
│         ECS Service Auto-Scaling Policy                     │
│                                                             │
│  Current: 10 tasks (2 vCPU, 4GB RAM each)                   │
│  Min: 10 tasks                                              │
│  Max: 200 tasks                                             │
│  Scale-out: +50%                                            │
│  Scale-in: -25%                                             │
│  Cooldown: 2 minutes                                        │
└──────────────────┬──────────────────────────────────────────┘
                   │
                   ▼
         ┌──────────────────┐
         │  Add 5 ECS Tasks │
         │                  │
         │  Duration: 60s   │ ← Fargate quick scaling
         └──────────────────┘
```

### Aurora Read Replica Auto-Scaling

```
Metric: CPU > 75% on readers OR Connections > 500

Current State:              Scale-Out (< 5 min):
┌──────────┐                ┌──────────┐
│  Writer  │                │  Writer  │
└────┬─────┘                └────┬─────┘
     │                           │
     ├─────┐                     ├─────┬───────┬───────┐
     │     │                     │     │       │       │
┌────▼┐ ┌─▼───┐             ┌────▼┐   ┌─▼───┐  ┌▼────┐ ┌▼────┐
│Read1│ │Read2│             │Read1│   │Read2│  │Read3│ │Read4│
│AZ-1a│ │AZ-1b│             │AZ-1a│   │AZ-1b│  │AZ-1c│ │AZ-1a│
└─────┘ └─────┘             └─────┘   └─────┘  └─────┘ └─────┘
                

2 Replicas → 4 Replicas (added 2 more)
```

---

## Cost Breakdown (Monthly)

### Baseline: 500 RPS

```
┌────────────────────────────────────────────────────────────┐
│ SERVICE               │ SPECS          │ QTY  │    COST    │
├────────────────────────────────────────────────────────────┤
│ API Gateway           │ 500 RPS        │      │   $1,050   │
│  - REST API           │ 12M req/month  │      │     $840   │
│  - WebSocket          │ 1M connections │      │     $210   │
│                       │                │      │            │
│ ECS Fargate           │ 2 vCPU, 4GB    │  10  │    $730    │
│  - Task compute       │ $73/task/month │      │            │
│                       │                │      │            │
│ Aurora PostgreSQL     │ db.r6g.large   │  3   │    $900    │
│  - Writer instance    │                │  1   │     $365   │
│  - Reader instances   │                │  2   │     $535   │
│                       │                │      │            │
│ ElastiCache Redis     │ cache.r6g.large│  2   │    $400    │
│  - Primary + Replica  │ 13GB RAM each  │      │            │
│                       │                │      │            │
│ DynamoDB              │ On-Demand mode │      │    $300    │
│  - 5M reads/month     │                │      │            │
│  - 2M writes/month    │                │      │            │
│                       │                │      │            │
│ Lambda                │ 1M invocations │      │     $50    │
│  - 256MB, 3s avg      │                │      │            │
│                       │                │      │            │
│ SQS FIFO              │ 10M requests   │      │     $40    │
│                       │                │      │            │
│ Kinesis Streams       │ 2 shards       │      │     $70    │
│                       │                │      │            │
│ S3                    │ 500GB storage  │      │     $15    │
│  - Standard tier      │                │      │            │
│                       │                │      │            │
│ CloudWatch            │ Logs + Metrics │      │    $150    │
│  - 50GB logs/month    │                │      │            │
│  - Custom metrics     │                │      │            │
│                       │                │      │            │
│ CloudFront            │ 1TB data       │      │     $85    │
│                       │                │      │            │
│ Misc Services         │ WAF, Cognito   │      │     $40    │
│  - Secrets Manager    │                │      │            │
│  - ACM, Route53       │                │      │            │
├────────────────────────────────────────────────────────────┤
│ TOTAL (BASELINE)      │                │      │  $3,830    │
│                       │                │      │            │
│ With Reserved:        │ -15% savings   │      │  $3,255    │
│ With Savings Plans:   │ -20% savings   │      │  $3,064    │
└────────────────────────────────────────────────────────────┘

Cost per sustained RPS (monthly): $7.66  ($3,830 / 500 RPS)
```

---

## Disaster Recovery Strategy

### Backup Configuration

```
┌─────────────────────────────────────────────────────────────┐
│  SERVICE          │ BACKUP METHOD        │ RETENTION        │
├─────────────────────────────────────────────────────────────┤
│  Aurora DB        │ Automated snapshots  │ 35 days          │
│                   │ Point-in-time (PITR) │ 5-minute RPO     │
│                   │ Cross-region copy    │ us-west-2        │
│                   │                      │                  │
│  ElastiCache      │ Daily snapshots      │ 7 days           │
│                   │ Manual snapshots     │ Infinite         │
│                   │                      │                  │
│  DynamoDB         │ Point-in-time (PITR) │ 35 days          │
│                   │ On-demand backups    │ Until deleted    │
│                   │                      │                  │
│  S3               │ Versioning enabled   │ 90 days          │
│                   │ Cross-region replica │ us-west-2        │
│                   │ Lifecycle → Glacier  │ After 90 days    │
│                   │                      │                  │
│  ECS Config       │ Infrastructure code  │ Git repository   │
│                   │ (Terraform/CDK)      │ Infinite         │
└─────────────────────────────────────────────────────────────┘
```

### Recovery Objectives

```
┌──────────────────────────────────────────────────────────────┐
│  SCENARIO                │    RTO    │    RPO    │  ACTION   │
├──────────────────────────────────────────────────────────────┤
│  Single AZ failure       │  < 1 min  │     0     │ Automatic │
│  Service degradation     │  < 30 sec │     0     │ Automatic │
│  Region failure          │  < 15 min │  < 5 min  │  Manual   │
│  Data corruption         │  < 1 hour │  5 min    │  Manual   │
│  Complete disaster       │  < 4 hours│  < 1 hour │  Manual   │
└──────────────────────────────────────────────────────────────┘
```

---

## Security Architecture

### Defense in Depth

```
┌─────────────────────────────────────────────────────────────┐
│                       Layer 1: Edge                         │
│                                                             │
│  ✓ CloudFront with AWS Shield (DDoS protection)             │
│  ✓ AWS WAF (SQL injection, XSS, rate limiting)              │
│  ✓ Geographic blocking (block malicious regions)            │
└──────────────────┬──────────────────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────────────────┐
│                    Layer 2: API Gateway                     │
│                                                             │
│  ✓ Throttling (10,000 RPS soft limit)                       │
│  ✓ API key validation                                       │
│  ✓ Request/response validation                              │
│  ✓ Cognito JWT authorization                                │
└──────────────────┬──────────────────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────────────────┐
│                   Layer 3: Network (VPC)                    │
│                                                             │
│  ✓ Private subnets (no direct internet access)              │
│  ✓ Security groups (least privilege)                        │
│  ✓ Network ACLs                                             │
│  ✓ VPC Flow Logs (traffic analysis)                         │
└──────────────────┬──────────────────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────────────────┐
│               Layer 4: Application (ECS)                    │
│                                                             │
│  ✓ IAM roles (task-specific permissions)                    │
│  ✓ Secrets Manager (credential rotation)                    │
│  ✓ Container scanning (ECR image scan)                      │
│  ✓ Read-only root filesystem                                │
└──────────────────┬──────────────────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────────────────┐
│                  Layer 5: Data                              │
│                                                             │
│  ✓ Encryption at rest (KMS)                                 │
│  ✓ Encryption in transit (TLS 1.3)                          │
│  ✓ Database encryption (Aurora)                             │
│  ✓ S3 bucket encryption + versioning                        │
└─────────────────────────────────────────────────────────────┘
```

### Compliance & Auditing

```
┌────────────────────────────────────────────────────────────┐
│  CloudTrail (Management Events)                            │
│   - Who did what, when, from where                         │
│   - API call logging                                       │
│   - 90-day retention → S3 for long-term                    │
│                                                            │
│  VPC Flow Logs                                             │
│   - Network traffic patterns                               │
│   - Identify unauthorized access attempts                  │
│                                                            │
│  AWS GuardDuty                                             │
│   - Threat detection (ML-based)                            │
│   - Compromised instance detection                         │
│   - Cryptocurrency mining detection                        │
│                                                            │
│  AWS Config                                                │
│   - Resource compliance tracking                           │
│   - Configuration change history                           │
│   - Automated remediation                                  │
└────────────────────────────────────────────────────────────┘
```

---
# Scaling Strategy

## Current Architecture Capacity

### Baseline Performance (500 RPS)

| Component | Current Capacity | Bottleneck Point |
|-----------|-----------------|------------------|
| **API Gateway** | 10,000 RPS (soft limit) | Can request increase to 100,000 RPS |
| **ECS Fargate** | 10 tasks × 50 RPS = 500 RPS | Task count (scales to 1,000+ tasks) |
| **ElastiCache Redis** | cache.r6g.large (10,000 ops/sec) | Network throughput (scales to r6g.4xlarge) |
| **Aurora PostgreSQL** | db.r6g.large + 2 replicas | Write capacity (scales to r6g.16xlarge) |
| **DynamoDB** | On-Demand mode (40,000 RCU/WCU) | Auto-scales to millions |
| **SQS FIFO** | 3,000 TPS per queue | Number of queues (partition by symbol) |
| **Lambda** | 1,000 concurrent executions | Account limit (can increase to 10,000+) |

**Current Cost**: ~$3,830/month (full service breakdown; see architecture_visual.md)
**Peak Capacity**: ~2,000 RPS before major changes needed

---

## Growth Phase 1: 10x Traffic (5,000 RPS)

### Timeline: 6-12 months

### Scaling Changes Required

#### 1. API Layer
- **API Gateway**: Request soft limit increase to 50,000 RPS (AWS support ticket)
- **CloudFront**: No changes needed (scales automatically)
- **WAF**: Add rate limiting per IP (max 1,000 req/5min)

#### 2. Compute Layer
```diff
- ECS Tasks: 10 tasks (2 vCPU, 4GB RAM each)
+ ECS Tasks: 100 tasks (4 vCPU, 8GB RAM each)
  Auto-scaling: Target 70% CPU utilization
```

**Change**: Scale from 10 → 100 tasks, upgrade instance size

#### 3. Cache Layer
```diff
- ElastiCache: 1x cache.r6g.large (13.07 GB RAM)
+ ElastiCache: 3-node cluster, cache.r6g.xlarge (26.32 GB RAM)
  Multi-AZ with automatic failover
```

**Change**: Cluster mode + larger instances for higher throughput

#### 4. Database Layer
```diff
- Aurora Writer: db.r6g.large (2 vCPU, 16 GB)
- Aurora Readers: 2x db.r6g.large
+ Aurora Writer: db.r6g.2xlarge (8 vCPU, 64 GB)
+ Aurora Readers: 5x db.r6g.xlarge (geographically distributed)
```

**Change**: Vertical scaling + more read replicas for read-heavy queries

#### 5. Queue Layer
```diff
- SQS FIFO: 1 queue (3,000 TPS)
+ SQS FIFO: 5 queues partitioned by symbol hash
  (15,000 aggregate TPS)
```

**Change**: Partition queues by trading pair for higher throughput

### Cost Projection: Phase 1

| Service | Current | Phase 1 (10x) | Delta |
|---------|--------:|---------------:|------:|
| API Gateway | $1,050 | $10,500 | +$9,450 |
| ECS Fargate | $730 | $14,600 | +$13,870 |
| ElastiCache | $400 | $1,800 | +$1,400 |
| Aurora | $900 | $3,200 | +$2,300 |
| DynamoDB | $300 | $1,800 | +$1,500 |
| CloudWatch | $150 | $600 | +$450 |
| Lambda + SQS + Kinesis + S3 + Misc | $300 | $500 | +$200 |
| **TOTAL** | **$3,830** | **$32,500** | **+$28,670** |

> Note: Phase 1 total is dominated by ECS Fargate ($14,600) and API Gateway ($10,500). Smaller services scale sub-linearly.

**Cost per RPS**: $3,830/500 = $7.66 → $32,500/5,000 = $6.50 (15% reduction per RPS)

### Key Optimizations for Phase 1

1. **Reserved Capacity**: Purchase Aurora and ElastiCache reserved instances (30-40% savings)
2. **Savings Plans**: Commit to ECS Fargate compute for 1 year (20% savings)
3. **S3 Lifecycle**: Move historical data >90 days to Glacier ($0.004/GB vs $0.023/GB)
4. **CloudFront**: Optimize cache hit ratio to reduce origin requests

**Optimized Phase 1 Cost**: ~$24,000/month (26% savings)

---

## Growth Phase 2: 100x Traffic (50,000 RPS)

### Timeline: 2-3 years

### Major Architectural Changes

#### 1. Multi-Region Deployment

**Problem**: Single region becomes bottleneck  
**Solution**: Deploy trading clusters in 3 regions

```
Primary Region: us-east-1 (40% traffic)
Secondary Region: eu-west-1 (35% traffic)
Tertiary Region: ap-southeast-1 (25% traffic)
```

**Benefits**:
- Reduced latency for global users (sub-50ms p99)
- Disaster recovery across regions
- Compliance with regional data residency

#### 2. Change Data Capture (CDC) for Cross-Region Sync

**Service**: AWS DMS (Database Migration Service) + Kinesis
- Real-time replication of Aurora writes across regions
- DynamoDB Global Tables for session data
- EventBridge cross-region event bus for trade events

#### 3. Transition to Event-Driven Architecture

```diff
- SQS FIFO Queues (15,000 TPS limit)
+ Amazon MSK (Managed Kafka) - 1,000,000+ TPS
  Topics: orders, trades, market-data, user-events
  Partitions: 50 per topic
```

**Rationale**: SQS FIFO cannot scale to 50,000 RPS economically

#### 4. Introduce API Caching Layer

```
CloudFront → API Gateway → Redis Cache → Origin
```

**Hit Ratio Target**: 80% for market data APIs (reduces backend load by 40,000 RPS)

#### 5. Database Sharding Strategy

**Horizontal Partitioning**:
- Shard users by `user_id % 10` → 10 Aurora clusters
- Each cluster handles 5,000 RPS
- Use Aurora Serverless v2 for auto-scaling within clusters

**Alternative**: Migrate to Amazon Aurora Limitless Database (preview) for transparent sharding

#### 6. Order Matching Engine Optimization

**Current**: General-purpose ECS containers  
**Phase 2**: Specialized compute

```diff
- ECS Fargate (general purpose)
+ ECS on EC2 with c7g instances (Graviton3)
  ARM-based, 40% better price/performance
  Enhanced networking (100 Gbps)
  + Dedicated matching engine in Rust/C++ (vs Node/Python)
```

**Performance Gain**: 5-10ms latency reduction per order

### Cost Projection: Phase 2

| Service | Phase 1 | Phase 2 (100x) | Delta |
|---------|--------:|----------------:|------:|
| API Gateway | $10,500 | $105,000 | +$94,500 |
| ECS on EC2 (reserved) | $14,600 | $80,000 | +$65,400 |
| MSK (Kafka) | $0 | $12,000 | +$12,000 |
| ElastiCache | $1,800 | $15,000 | +$13,200 |
| Aurora (10 shards) | $3,200 | $42,000 | +$38,800 |
| DynamoDB Global | $1,800 | $18,000 | +$16,200 |
| Data Transfer (multi-region) | $500 | $8,000 | +$7,500 |
| CloudWatch/X-Ray | $600 | $3,000 | +$2,400 |
| **TOTAL** | **$32,500** | **$283,000** | **+$250,500** |

**Cost per RPS**: $32,500/5,000 = $6.50 → $283,000/50,000 = $5.66 (13% reduction per RPS)

### Phase 2 Optimizations

1. **Compute Savings Plans**: 3-year commitment on EC2 (50% savings)
2. **Reserved Aurora Instances**: 3-year all-upfront (60% savings)
3. **S3 Intelligent Tiering**: Automatic cost optimization for historical data
4. **API Gateway HTTP API**: Migrate from REST to HTTP API (70% cost reduction)
5. **PrivateLink**: Reduce data transfer costs between VPCs

**Optimized Phase 2 Cost**: ~$180,000/month (36% savings)

---

## Scaling Decision Tree

```
                        +-------------------+
                        |  Current: 500 RPS |
                        +-------------------+
                                 |
                                 v
                   +---------------------------+
                   |   Traffic > 2,000 RPS?    |
                   +---------------------------+
                    No |                 | Yes
                       v                 v
            +-------------------+   +---------------------------+
            | Vertical Scaling  |   |   Traffic > 5,000 RPS?    |
            | Larger Instances  |   +---------------------------+
            +-------------------+    No |                 | Yes
                       |                v                 v
                       |  +----------------------+  +---------------------------+
                       |  | Horizontal Scaling   |  |  Traffic > 20,000 RPS?   |
                       |  | More Tasks/Replicas  |  +---------------------------+
                       |  +----------------------+   No |                | Yes
                       |               |                v                v
                       |               |  +------------------+  +---------------------------+
                       |               |  | Partition Queues |  |  Traffic > 50,000 RPS?   |
                       |               |  | by Symbol        |  +---------------------------+
                       |               |  +------------------+   No |               | Yes
                       |               |               |            v               v
                       |               |               |  +-----------------+  +------------------+
                       |               |               |  | Multi-Region    |  | Migrate to MSK   |
                       |               |               |  | Deployment      |  | + DB Sharding    |
                       |               |               |  +-----------------+  +------------------+
                       |               |               |              |               |
                       +---------------+---------------+--------------+---------------+
                                                       |
                                                       v
                                            +--------------------+
                                            |  Monitor & Optimize|
                                            +--------------------+
                                                       |
                                                       v (loop back)
                                            +-------------------+
                                            |  Current: 500 RPS |
                                            +-------------------+
```

---

## Monitoring & Triggers for Scaling

### Auto-Scaling Triggers

| Metric | Threshold | Action |
|--------|-----------|--------|
| **CPU Utilization** | > 70% for 5 min | Add ECS tasks (scale out) |
| **Memory Utilization** | > 80% for 5 min | Add ECS tasks |
| **API Gateway 4xx Rate** | > 5% | Investigate throttling, scale out |
| **API Gateway Latency (p99)** | > 80ms | Scale out ECS + Aurora replicas |
| **Aurora CPU** | > 75% | Add read replicas or scale up |
| **DynamoDB Throttled Requests** | > 0 | Increase provisioned capacity |
| **SQS Queue Depth** | > 1,000 messages | Add Lambda concurrency |
| **ElastiCache Evictions** | > 100/min | Upgrade instance size |

### Predictive Scaling

Use **AWS Auto Scaling Predictive Scaling** with ML models:
- Analyze historical traffic patterns
- Pre-scale before known peaks (market open, major announcements)
- 24-hour forecast window

### Cost Alarms

- **Daily Cost > $150**: Alert DevOps (expected: $120/day)
- **Weekly Cost > $1,200**: Alert Finance
- **DynamoDB On-Demand > $100/day**: Consider provisioned capacity

---

## Disaster Recovery & Scaling for Failures

### RTO/RPO Targets

| Scenario | RTO (Recovery Time Objective) | RPO (Recovery Point Objective) |
|----------|------------------------------|-------------------------------|
| AZ Failure | < 1 minute (automatic) | 0 (Multi-AZ sync) |
| Region Failure | < 15 minutes (manual) | < 5 minutes (async replication) |
| Service Degradation | < 30 seconds (auto-scaling) | N/A |
| Data Corruption | < 1 hour (restore from backup) | 5 minutes (PITR) |

### Failure Handling Capacity

**Scenario**: Sudden 10x traffic spike (e.g., market crash)

1. **API Gateway**: Throttle to 10,000 RPS (protects backend)
2. **ECS Auto-Scaling**: Scale from 10 → 200 tasks in 5 minutes
3. **Aurora**: Read replicas auto-scale in 3-5 minutes
4. **DynamoDB**: Auto-scales instantly (On-Demand mode)
5. **SQS**: Buffers requests (unlimited queue depth)

**Result**: System degrades gracefully, no hard failures

---

## Summary: Scaling Roadmap

| Phase | Traffic | Timeline | Key Changes | Monthly Cost |
|-------|---------|----------|-------------|-------------:|
| **Current** | 500 RPS | Now | Baseline architecture | $3,580 |
| **Phase 1** | 5,000 RPS | 6-12 months | More tasks, larger instances | $24,000 |
| **Phase 2** | 50,000 RPS | 2-3 years | Multi-region, MSK, sharding | $180,000 |

**Key Principle**: Scale incrementally, optimize continuously, avoid premature optimization.

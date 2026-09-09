# 04. AWS Cloud Architecture, VPC Networking & Production Infrastructure

> **Target Role**: Staff / Senior Backend & Infrastructure Engineer  
> **Module**: 08-docker-devops-aws / 04-aws-cloud-architecture  
> **Key Focus**: Compute Primitives (EC2, EBS, ECS Fargate vs EC2, Lambda), VPC Architecture & Subnet Isolation, Security Group Chaining, Route 53 & Load Balancing (ALB vs NLB), S3 Storage & Pre-signed URLs, RDS Multi-AZ vs Read Replicas, ElastiCache Redis, SQS/SNS Messaging, and CloudWatch/X-Ray Observability.

---

## Table of Contents
1. [Compute Primitives & Container Orchestration (EC2, ECS & Lambda)](#1-compute-primitives--container-orchestration-ec2-ecs--lambda)
   - [1.1 Definition & Core Concept](#11-definition--core-concept)
   - [1.2 Internal Mechanics: ECS Agent, Nitro Hypervisor & IAM Roles](#12-internal-mechanics-ecs-agent-nitro-hypervisor--iam-roles)
   - [1.3 Production ECS Fargate Task Definition & Terraform Specs](#13-production-ecs-fargate-task-definition--terraform-specs)
   - [1.4 Production Outages & Debugging](#14-production-outages--debugging)
   - [1.5 Trade-offs & Decision Matrix](#15-trade-offs--decision-matrix)
   - [1.6 Senior Interview Q&A](#16-senior-interview-qa)
2. [VPC Networking, Subnet Topology & Security Group Chaining](#2-vpc-networking-subnet-topology--security-group-chaining)
   - [2.1 Definition & Core Concept](#21-definition--core-concept)
   - [2.2 Internal Mechanics: Subnet Route Tables, NAT Gateways & Packet Routing](#22-internal-mechanics-subnet-route-tables-nat-gateways--packet-routing)
   - [2.3 Production Terraform VPC & Security Group Rules](#23-production-terraform-vpc--security-group-rules)
   - [2.4 Production Outages & Debugging](#24-production-outages--debugging)
   - [2.5 Trade-offs & Decision Matrix](#25-trade-offs--decision-matrix)
   - [2.6 Senior Interview Q&A](#26-senior-interview-qa)
3. [S3 Object Storage Engineering & Security](#3-s3-object-storage-engineering--security)
   - [3.1 Definition & Core Concept](#31-definition--core-concept)
   - [3.2 Internal Mechanics: Distributed Prefix Sharding, Strong Consistency & Pre-Signed URLs](#32-internal-mechanics-distributed-prefix-sharding-strong-consistency--pre-signed-urls)
   - [3.3 Production Node.js & Ruby Pre-Signed URL Generators](#33-production-nodejs--ruby-pre-signed-url-generators)
   - [3.4 Production Outages & Debugging](#34-production-outages--debugging)
   - [3.5 Trade-offs & Decision Matrix](#35-trade-offs--decision-matrix)
   - [3.6 Senior Interview Q&A](#36-senior-interview-qa)
4. [Managed Databases & Caching (RDS & ElastiCache)](#4-managed-databases--caching-rds--elasticache)
   - [4.1 Definition & Core Concept](#41-definition--core-concept)
   - [4.2 Internal Mechanics: Multi-AZ Synchronous Replication vs. Asynchronous Read Replicas](#42-internal-mechanics-multi-az-synchronous-replication-vs-asynchronous-read-replicas)
   - [4.3 Production Database Architectures & Failover Scripts](#43-production-database-architectures--failover-scripts)
   - [4.4 Production Outages & Debugging](#44-production-outages--debugging)
   - [4.5 Trade-offs & Decision Matrix](#45-trade-offs--decision-matrix)
   - [4.6 Senior Interview Q&A](#46-senior-interview-qa)
5. [Distributed Messaging & Observability (SQS, SNS, CloudWatch & X-Ray)](#5-distributed-messaging--observability-sqs-sns-cloudwatch--x-ray)
   - [5.1 Definition & Core Concept](#51-definition--core-concept)
   - [5.2 Internal Mechanics: Standard vs FIFO Queues, DLQ Redrive & X-Ray Trace Context](#52-internal-mechanics-standard-vs-fifo-queues-dlq-redrive--x-ray-trace-context)
   - [5.3 Production Fanout Topology & Worker Implementation](#53-production-fanout-topology--worker-implementation)
   - [5.4 Production Outages & Debugging](#54-production-outages--debugging)
   - [5.5 Trade-offs & Decision Matrix](#55-trade-offs--decision-matrix)
   - [5.6 Senior Interview Q&A](#56-senior-interview-qa)

---

# 1. Compute Primitives & Container Orchestration (EC2, ECS & Lambda)

### 1.1 Definition & Core Concept
AWS provides compute across three primary abstractions:
- **EC2 (Elastic Compute Cloud)**: Virtual machines running on the AWS Nitro hypervisor. Sized by workload (Compute Optimized `c7g`, Memory Optimized `r7g`, General Purpose `m7g`). Storage backed by **EBS (Elastic Block Store)** (`gp3` for baseline workloads, `io2 Block Express` for sub-millisecond, mission-critical database storage).
- **AWS ECS (Elastic Container Service)**: A fully managed container orchestration platform.
  - **ECS EC2**: Containers run on user-managed EC2 instances. Full host control, lower pure compute cost, but requires OS patching, AMI maintenance, and cluster capacity planning.
  - **ECS Fargate**: Serverless container execution. AWS dynamically manages the underlying host compute, OS, and security patches. Pay strictly for vCPU and memory allocated per second per task.
- **AWS Lambda**: Serverless event-driven compute executing short-lived functions ($<15\text{ minutes}$) in response to triggers.

---

### 1.2 Internal Mechanics: ECS Agent, Nitro Hypervisor & IAM Roles

#### 1. Task Execution Role vs. Task Role (IAM Least Privilege)
This is an essential security distinction in ECS:

```
+---------------------------------------------------------------------------------+
|                               AWS ECS TASK                                      |
|                                                                                 |
|   +-------------------------------------------------------------------------+   |
|   | 1. ECS TASK EXECUTION ROLE (Used by AWS ECS Infrastructure Agent)       |   |
|   |    - ecr:GetAuthorizationToken / ecr:BatchGetImage (Pull images)        |   |
|   |    - logs:CreateLogStream / logs:PutLogEvents (Write to CloudWatch)     |   |
|   |    - secretsmanager:GetSecretValue (Decrypt secrets to inject into env) |   |
|   +-------------------------------------------------------------------------+   |
|                                       │                                         |
|                                       ▼                                         |
|   +-------------------------------------------------------------------------+   |
|   | 2. ECS TASK ROLE (Assumed by YOUR Application Code via AWS Metadata)     |   |
|   |    - s3:PutObject / s3:GetObject (Access specific S3 bucket)            |   |
|   |    - sqs:SendMessage / sqs:ReceiveMessage (Process queues)             |   |
|   |    - ZERO EC2/ECR permissions; cannot alter infrastructure              |   |
|   +-------------------------------------------------------------------------+   |
+---------------------------------------------------------------------------------+
```

- **Task Execution Role**: Grants permissions to the **AWS ECS container agent** running on the host before your container starts.
- **Task Role**: Grants permissions to the **actual application running inside the container**. When the app initializes the AWS SDK (`new AWS.S3()`), the SDK queries the local ECS metadata endpoint (`169.254.170.2$AWS_CONTAINER_CREDENTIALS_RELATIVE_URI`) to retrieve temporary STS credentials associated with the Task Role.

#### 2. AWS Lambda Execution Environment & Cold Starts
When a Lambda function is invoked:
1. **Cold Start Phase**:
   - **Init Phase**: AWS provisions a lightweight MicroVM (via Firecracker).
   - Downloads the container image/zip package.
   - Bootstraps the language runtime (Node.js/Ruby).
   - Executes global initialization code outside the handler (DB connection setup, SDK instantiations).
   - Duration: 200ms - 3,000ms.
2. **Warm Execution Phase**:
   - The MicroVM is frozen in memory between invocations. Subsequent requests reuse the warm environment with near-zero latency (<5ms).
3. **Concurrency Limits**:
   - Account limit default: 1,000 concurrent executions per region. Exceeding concurrency triggers HTTP 429 Too Many Requests (`ProvisionedConcurrency` eliminates cold starts for predictable workloads).

---

### 1.3 Production ECS Fargate Task Definition & Terraform Specs

```json
{
  "family": "production-api-task",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "1024",
  "memory": "2048",
  "executionRoleArn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::123456789012:role/apiApplicationTaskRole",
  "containerDefinitions": [
    {
      "name": "api-container",
      "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/api:c4f1e0a",
      "essential": true,
      "portMappings": [
        {
          "containerPort": 3000,
          "protocol": "tcp"
        }
      ],
      "secrets": [
        {
          "name": "DATABASE_URL",
          "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789012:secret:prod/db-url"
        }
      ],
      "environment": [
        { "name": "NODE_ENV", "value": "production" },
        { "name": "PORT", "value": "3000" }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/production-api",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      },
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:3000/health/live || exit 1"],
        "interval": 15,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 30
      }
    }
  ]
}
```

---

### 1.4 Production Outages & Debugging

#### Outage Case Study: EBS `gp3` IOPS Starvation Under Heavy Database Writes
- **Context**: A PostgreSQL database running on an EC2 instance (`m6i.2xlarge`) with a 500 GB `gp3` EBS volume ground to a halt during an end-of-month financial reconciliation run. Write queries took 14 seconds.
- **Root Cause**: `gp3` provides 3,000 baseline IOPS and 125 MB/s throughput regardless of volume size. During the batch run, write operations spiked to 7,500 IOPS. Because provisioned IOPS had not been purchased above baseline, EBS aggressively throttled I/O. Linux processes entered `D` (Uninterruptible Sleep) state waiting on disk block flushes (`iowait` hit 88%).
- **Resolution**:
  1. Modified the EBS volume via AWS CLI online (zero downtime!):
     ```bash
     aws ec2 modify-volume \
       --volume-id vol-0abc123def456 \
       --iops 10000 \
       --throughput 500
     ```
  2. EBS dynamic volume modification expanded throughput within 60 seconds without unmounting the filesystem.

---

### 1.5 Trade-offs & Decision Matrix

| Compute Model | Operational Overhead | Cold Start Latency | Scaling Speed | Ideal Workload |
|:---|:---|:---|:---|:---|
| **EC2 Auto Scaling** | High (OS hardening, AMI updates) | None (Pre-warmed) | Minutes (EC2 boot time) | Steady-state monolithic systems, custom kernel modules. |
| **ECS Fargate** | **Near Zero** (Serverless containers) | 15-45 seconds (Task launch) | Fast (Task parallel boot) | **Standard for modern web APIs, microservices, background queues**. |
| **AWS Lambda** | Zero | 200ms - 2s (Cold starts) | **Milliseconds** (Massive burst) | Sporadic jobs, webhooks, S3 image processing, ETL triggers. |

---

### 1.6 Senior Interview Q&A

#### Q: In ECS Fargate, why must task networking always use `awsvpc` mode, and what impact does this have on VPC IP address planning?
**Answer**:
In ECS Fargate, AWS enforces `awsvpc` network mode because each Fargate task is allocated its own dedicated **Elastic Network Interface (ENI)** with a private IP directly from your VPC subnet.
- **Security Impact**: Security Groups can be assigned directly to individual tasks, providing micro-segmentation identical to EC2 instances.
- **IP Address Planning Hazard**: If a service runs 50 Fargate tasks across 2 private subnets, it consumes 50 distinct private IP addresses. If your VPC subnet was created with a small CIDR block (e.g., `/24` with 251 usable IPs), scaling up tasks alongside RDS, Redis, and NAT Gateways will **exhaust the subnet's IP space**, causing new task launches to fail with `ResourceInitializationError: unable to allocate ENI`. Subnets hosting autoscaling Fargate workloads must be provisioned with at least a `/20` or `/19` CIDR block.

---

# 2. VPC Networking, Subnet Topology & Security Group Chaining

### 2.1 Definition & Core Concept
A **Virtual Private Cloud (VPC)** is an isolated software-defined virtual network spanning an AWS region. Production VPCs strictly divide workloads into:
- **Public Subnets**: Have a direct route to an **Internet Gateway (IGW)**. Only internet-facing Application Load Balancers (ALBs) or bastion hosts may reside here.
- **Private Subnets**: Have no direct route to the internet. Outbound internet access (e.g., pulling npm packages, calling Stripe API) is routed through a managed **NAT Gateway** residing in the public subnet. All application workloads (ECS, EC2, Lambda) live here.
- **Isolated (Database) Subnets**: Have **ZERO route to the internet** (no IGW, no NAT Gateway). RDS, Aurora, and ElastiCache clusters live here, completely unroutable from the public internet.

```
                                  INTERNET
                                     │
                                     ▼
                          +----------------------+
                          |   Internet Gateway   |
                          +----------------------+
                                     │
==================================== VPC ====================================
PUBLIC SUBNET (10.0.1.0/24)          │
  ┌──────────────────────────────────┴──────────────────────────────────┐
  │ Application Load Balancer (ALB) [Security Group: alb-sg (0.0.0.0/0)]│
  │ NAT Gateway (Elastic IP: 54.21.9.12)                                │
  └──────────────────────────────────┬──────────────────────────────────┘
                                     │
PRIVATE SUBNET (10.0.10.0/24)        │ (Outbound to NAT / Inbound from ALB)
  ┌──────────────────────────────────▼──────────────────────────────────┐
  │ ECS Fargate Tasks (Web / Worker)                                    │
  │ [Security Group: app-sg (Allows port 3000 ONLY from alb-sg ID)]     │
  └──────────────────────────────────┬──────────────────────────────────┘
                                     │
ISOLATED SUBNET (10.0.20.0/24)       │ (Internal Traffic ONLY)
  ┌──────────────────────────────────▼──────────────────────────────────┐
  │ Amazon RDS PostgreSQL & ElastiCache Redis                           │
  │ [Security Group: db-sg (Allows port 5432 ONLY from app-sg ID)]      │
  │ ZERO ROUTE TO INTERNET (Immune to external attacks)                 │
  └─────────────────────────────────────────────────────────────────────┘
```

---

### 2.2 Internal Mechanics: Subnet Route Tables, NAT Gateways & Packet Routing

#### 1. Security Group Chaining (Least Privilege Defense-in-Depth)
**The Anti-Pattern**: Allowing inbound database traffic from a CIDR block:
`Ingress: Port 5432, CIDR 10.0.0.0/16` (Allows ANY compromised container or staging service in the VPC to connect to production DB).

**The Production Pattern (Security Group ID Referencing)**:
1. `alb-sg`: Allows inbound `80` and `443` from `0.0.0.0/0`.
2. `app-sg`: Inbound port `3000` is permitted **ONLY if the source is `alb-sg`**.
3. `db-sg`: Inbound port `5432` is permitted **ONLY if the source is `app-sg`**.
If an attacker exploits a remote code execution vulnerability on an EC2 instance inside the private subnet, but that instance does not hold the `app-sg` Security Group ID, the AWS hypervisor drops database packets at the virtual switch layer before reaching RDS.

#### 2. Route 53 & Load Balancer Topologies: ALB vs. NLB
- **Route 53**: Global DNS service providing **Latency-based routing** (routes users to lowest latency region), **Geolocation routing** (regulatory data boundaries), and **DNS Failover** (health checks automatically repoint domain if a region goes down).
- **Application Load Balancer (ALB)**: Operates at **Layer 7**. Inspects HTTP headers, paths (`/api` -> Service A, `/checkout` -> Service B), handles TLS termination with ACM, and injects `X-Forwarded-For`.
- **Network Load Balancer (NLB)**: Operates at **Layer 4**. Handles tens of millions of requests per second with ultra-low latency (<1ms). Routes raw TCP/UDP packets. Provides static Elastic IPs per AZ (essential for client whitelist firewalls).

---

### 2.3 Production Terraform VPC & Security Group Rules

```hcl
# ------------------------------------------------------------------------------
# Security Group: Application Load Balancer (Public Ingress)
# ------------------------------------------------------------------------------
resource "aws_security_group" "alb" {
  name        = "production-alb-sg"
  description = "Allow inbound HTTPS from internet"
  vpc_id      = aws_vpc.main.id

  ingress {
    description = "Allow TLS from anywhere"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# ------------------------------------------------------------------------------
# Security Group: ECS Application Tasks (Private Subnet)
# ------------------------------------------------------------------------------
resource "aws_security_group" "app" {
  name        = "production-app-sg"
  description = "Allow inbound traffic exclusively from ALB"
  vpc_id      = aws_vpc.main.id

  ingress {
    description     = "Allow port 3000 ONLY from ALB Security Group"
    from_port       = 3000
    to_port         = 3000
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id] # CHAINED REFERENCE!
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"] # Reaches internet via NAT Gateway
  }
}

# ------------------------------------------------------------------------------
# Security Group: PostgreSQL Database (Isolated Subnet)
# ------------------------------------------------------------------------------
resource "aws_security_group" "db" {
  name        = "production-db-sg"
  description = "Allow inbound PostgreSQL exclusively from Application Tasks"
  vpc_id      = aws_vpc.main.id

  ingress {
    description     = "PostgreSQL port 5432 ONLY from App SG"
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id] # CHAINED REFERENCE!
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = [] # No outbound internet access allowed
  }
}
```

---

### 2.4 Production Outages & Debugging

#### Outage Case Study: The Single NAT Gateway Multi-AZ Blackout
- **Context**: An enterprise deployed ECS tasks across three Availability Zones (`us-east-1a`, `us-east-1b`, `us-east-1c`) for high availability.
- **Outage**: When `us-east-1a` suffered a localized power outage, services across **all three AZs** completely lost internet connectivity and failed health checks.
- **Root Cause**: To save costs, the infrastructure team provisioned only **one NAT Gateway** located in `us-east-1a`, and pointed private subnets in `1b` and `1c` to route through that single NAT Gateway. When `1a` failed, outbound internet traffic for the entire cluster died.
- **Resolution**: Provisioned a **dedicated NAT Gateway per Availability Zone** (3 NAT Gateways total), ensuring complete architectural failure isolation.

---

### 2.5 Trade-offs & Decision Matrix

| Load Balancer | OSI Layer | Latency | SSL Termination | Static IP Support | Best Fit |
|:---|:---|:---|:---|:---|:---|
| **ALB** | Layer 7 (HTTP/S) | 4-12ms | Yes (ACM certificates) | No (Dynamic DNS name) | Web applications, REST APIs, microservice path routing. |
| **NLB** | Layer 4 (TCP/UDP) | < 1ms | Yes (High performance) | **Yes (1 Elastic IP per AZ)** | Financial protocols, gaming, raw TCP, legacy whitelists. |

---

### 2.6 Senior Interview Q&A

#### Q: What is the difference between a Security Group and a Network Access Control List (NACL) in AWS?
**Answer**:

| Attribute | Security Group | Network ACL (NACL) |
|:---|:---|:---|
| **Operating Layer** | Instance / ENI level (Virtual Firewall) | Subnet level (Boundary Firewall) |
| **State Tracking** | **Stateful** (Inbound return traffic automatically allowed) | **Stateless** (Must explicitly permit outbound ephemeral ports 1024-65535) |
| **Rule Evaluation** | Evaluates all rules before deciding | Evaluates rules sequentially in numerical order (Rule 100, 200...) |
| **Deny Capabilities** | Permits allow rules only | Supports both **ALLOW** and **DENY** rules (Essential for blocking malicious IPs) |

---

# 3. S3 Object Storage Engineering & Security

### 3.1 Definition & Core Concept
**Amazon Simple Storage Service (S3)** is an exabyte-scale, highly available (99.99%) and durable (99.999999999% - 11 9's) object store:
- **Strong Consistency**: S3 provides strong read-after-write consistency for `PUT` and `DELETE` requests of objects in all regions with zero latency penalty.
- **Pre-Signed URLs**: Cryptographically signed URLs allowing clients (browsers/mobile apps) to upload (`PUT`) or download (`GET`) directly to/from S3 without exposing application backend servers to high-bandwidth file proxying or leaking master AWS credentials.

---

### 3.2 Internal Mechanics: Distributed Prefix Sharding, Strong Consistency & Pre-Signed URLs

#### 1. S3 Request Partitioning & Performance Limits
S3 scales horizontally across storage partitions.
- Each partition supports **3,500 PUT/POST/DELETE requests per second** and **5,500 GET/HEAD requests per second**.
- Partitioning is determined by the **Object Key Prefix** (e.g., `s3://bucket/prefix/object.jpg`).
- Modern S3 automatically shards partitions behind the scenes; prefix randomization (MD5 hashing prefixes) is no longer required unless exceeding 100,000 req/sec.

#### 2. Storage Lifecycle Management (Cost Optimization Engine)
Automated tiering migrates aging objects down storage classes:
- **S3 Standard**: Active hot data accessed frequently.
- **S3 Standard-IA (Infrequent Access)**: Cheaper storage; rapid retrieval fee. (For data accessed < once a month).
- **S3 Glacier Flexible Retrieval / Deep Archive**: Ultra-low cost long-term cold storage ($1 per TB/month). Retrieval takes minutes to hours.

```
[Day 0: User Upload] ────► S3 Standard ($0.023/GB/mo)
                                 │
                                 │ (Transition after 30 days)
                                 ▼
                           S3 Infrequent Access ($0.0125/GB/mo)
                                 │
                                 │ (Transition after 90 days)
                                 ▼
                           S3 Glacier Deep Archive ($0.00099/GB/mo)
                                 │
                                 │ (Expire after 365 days)
                                 ▼
                           PERMANENT DELETION (Free)
```

---

### 3.3 Production Node.js & Ruby Pre-Signed URL Generators

#### 1. TypeScript Direct-to-S3 Upload Generator
```typescript
import { S3Client, PutObjectCommand, GetObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';

const s3 = new S3Client({ region: 'us-east-1' });
const BUCKET_NAME = process.env.USER_UPLOADS_BUCKET!;

// Generate temporary PUT URL allowing client browser to upload directly to S3
export async function generateUploadUrl(userId: string, filename: string, contentType: string): Promise<string> {
  const objectKey = `uploads/${userId}/${Date.now()}-${filename}`;

  const command = new PutObjectCommand({
    Bucket: BUCKET_NAME,
    Key: objectKey,
    ContentType: contentType,
  });

  // URL valid for exactly 15 minutes
  return await getSignedUrl(s3, command, { expiresIn: 900 });
}

// Generate temporary secure GET URL
export async function generateDownloadUrl(objectKey: string): Promise<string> {
  const command = new GetObjectCommand({
    Bucket: BUCKET_NAME,
    Key: objectKey,
  });

  return await getSignedUrl(s3, command, { expiresIn: 300 });
}
```

#### 2. Production S3 Bucket Policy (Restricting to CloudFront OAI/OAC)
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCloudFrontServicePrincipalReadOnly",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudfront.amazonaws.com"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::production-static-assets/*",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "arn:aws:cloudfront::123456789012:distribution/E1A2B3C4D5E6F7"
        }
      }
    }
  ]
}
```

---

### 3.4 Production Outages & Debugging

#### Outage Case Study: Backend OOM Caused by File Proxying
- **Context**: A Node.js backend offered a file upload endpoint `/api/documents`. When users uploaded 500 MB video files, Node.js buffered chunks into heap memory.
- **Outage**: During high traffic, 10 concurrent uploads consumed 5 GB of RAM, triggering Node.js V8 heap allocation crashes and killing the API.
- **Resolution**: Eliminated backend proxying by migrating to **S3 Pre-Signed PUT URLs**. The frontend asks the backend for a signature (JSON response in 5ms), and then uploads the 500 MB payload directly to S3 via HTTP PUT. Zero application server bandwidth or memory is consumed.

---

### 3.5 Trade-offs & Decision Matrix

| S3 Class | Storage Cost (per GB/mo) | Retrieval Fee | Minimum Storage Duration | Best Used For |
|:---|:---|:---|:---|:---|
| **S3 Standard** | ~$0.023 | None | None | Live web assets, dynamic user profile pictures. |
| **S3 Standard-IA** | ~$0.0125 | ~$0.01 / GB | 30 days | Invoices, monthly bank statements. |
| **S3 Glacier Deep Archive** | ~$0.00099 | ~$0.02 / GB | 180 days | Legal compliance records, annual DB backup archives. |

---

### 3.6 Senior Interview Q&A

#### Q: Why is S3 CORS configuration necessary when using Pre-Signed URLs, and what headers must be allowed?
**Answer**:
When a browser uploads directly to S3 from `https://app.enterprise.com`, the browser executes a cross-origin HTTP request to `https://<bucket>.s3.amazonaws.com`.
The browser dispatches a pre-flight `OPTIONS` request. If S3 lacks a valid CORS policy matching the origin, the browser aborts the request before uploading.
The S3 bucket must have a CORS rule configured:
```xml
<CORSRule>
    <AllowedOrigin>https://app.enterprise.com</AllowedOrigin>
    <AllowedMethod>PUT</AllowedMethod>
    <AllowedMethod>GET</AllowedMethod>
    <AllowedHeader>*</AllowedHeader>
    <MaxAgeSeconds>3000</MaxAgeSeconds>
</CORSRule>
```

---

# 4. Managed Databases & Caching (RDS & ElastiCache)

### 4.1 Definition & Core Concept
- **Amazon RDS (Relational Database Service)**: Managed relational databases (PostgreSQL, MySQL).
  - **Multi-AZ Deployment**: High-availability configuration featuring **synchronous physical block-level replication** to a standby instance in a different AZ.
  - **Read Replicas**: Scalability configuration using **asynchronous replication** to offload read-heavy SQL queries.
- **Amazon Aurora**: Cloud-native relational database decoupling compute from a distributed 6-way replicated storage fleet across 3 AZs.
- **Amazon ElastiCache (Redis)**: In-memory distributed caching service featuring clustering, replication groups, and automated failover.

---

### 4.2 Internal Mechanics: Multi-AZ Synchronous Replication vs. Asynchronous Read Replicas

```
+---------------------------------------------------------------------------------+
|                       AWS REGION (e.g., us-east-1)                              |
|                                                                                 |
|   AVAILABILITY ZONE A                     AVAILABILITY ZONE B                   |
|   ┌───────────────────────────┐           ┌───────────────────────────┐         |
|   │ PRIMARY DB (Read/Write)   │           │ STANDBY DB (Failover Target)        |
|   │ Writes transaction to WAL │           │ Hidden; Inaccessible to queries   |
|   └─────────────┬─────────────┘           └─────────────▲─────────────┘         |
|                 │                                       │                       |
|                 │ Synchronous EBS Block-Level Mirroring │                       |
|                 └───────────────────────────────────────┘                       |
|                                                                                 |
|                                           AVAILABILITY ZONE C                   |
|                                           ┌───────────────────────────┐         |
|   Asynchronous WAL Streaming Replication  │ READ REPLICA (Read-Only)  │         |
|   ───────────────────────────────────────►│ Serves BI/Analytical Qs   │         |
|   (Potential Replication Lag: 50-500ms)   │ Open for direct SQL SELECT│         |
|                                           └───────────────────────────┘         |
+---------------------------------------------------------------------------------+
```

#### Synchronous Multi-AZ Failover Timeline:
1. Primary instance experiences hardware failure or loss of AZ.
2. RDS detects heartbeat loss within 30 seconds.
3. Standby instance in AZ-B is promoted to Primary.
4. **RDS automatically flips the CNAME DNS record** of the database endpoint to point to the new Primary IP.
5. Failover completes in 60-120 seconds with **ZERO data loss** ($RPO = 0$).

---

### 4.3 Production Database Architectures & Failover Scripts

#### Rails Dual Database Configuration (`config/database.yml`)
```yaml
production:
  primary:
    adapter: postgresql
    url: <%= ENV['DATABASE_PRIMARY_URL'] %> # Points to RDS Multi-AZ Primary
    pool: 20
    timeout: 5000
  primary_replica:
    adapter: postgresql
    url: <%= ENV['DATABASE_REPLICA_URL'] %> # Points to Read Replica Endpoint
    pool: 20
    timeout: 5000
    replica: true
```

#### Application Auto-Routing (Rails Active Record):
```ruby
class ApplicationController < ActionController::Base
  # Automatically routes GET requests to Replica and POST/PUT/DELETE to Primary
  around_action :set_reading_role

  private

  def set_reading_role(&block)
    if request.get?
      ActiveRecord::Base.connected_to(role: :reading) { yield }
    else
      ActiveRecord::Base.connected_to(role: :writing) { yield }
    end
  end
end
```

---

### 4.4 Production Outages & Debugging

#### Outage Case Study: Read-After-Write Consistency Replication Lag
- **Context**: A user updated their shipping address on `/profile`, but on refresh, the page still displayed the old address. 2 seconds later, the address appeared correctly.
- **Root Cause**: The update executed against the RDS Primary via `POST`. The subsequent `GET` query was routed to the Read Replica. The replica suffered a 1,200ms **Replication Lag** (`pg_stat_replication`), returning stale snapshot data.
- **Resolution**: Implemented **Sticky Read-Your-Own-Writes**:
  When a user performs a write operation, set a short-lived cookie (`just_modified=true`, 3s TTL). If the cookie is present, route subsequent `GET` requests directly to the Primary database.

---

### 4.5 Trade-offs & Decision Matrix

| Configuration | Primary Purpose | Replication Mode | Failover Mechanism | Data Loss Risk (RPO) |
|:---|:---|:---|:---|:---|
| **Multi-AZ Standby** | High Availability / Disaster Recovery | **Synchronous** | Automatic DNS failover | **Zero ($RPO = 0$)** |
| **Read Replica** | Read Horizontal Scalability | **Asynchronous** | Manual promotion | Data loss possible if primary dies |
| **Aurora Multi-Master** | Continuous write availability | Storage quorum (4 of 6) | Instant (<100ms) | Zero |

---

### 4.6 Senior Interview Q&A

#### Q: How does Amazon Aurora storage differ from traditional RDS PostgreSQL storage architecture?
**Answer**:
In standard RDS, database compute instances write pages and WAL records to attached EBS volumes over the network. Multi-AZ mirrors every block write across EBS volumes, causing write amplification and EBS I/O bottlenecks.
**Amazon Aurora completely decouples compute from storage**:
1. Aurora instances write **ONLY the log stream (WAL records)** across the network to a purpose-built distributed storage fleet.
2. The storage fleet spans **3 Availability Zones, replicating 6 copies of each data block** (2 copies per AZ).
3. Writes achieve consensus via a **4-of-6 quorum**, meaning write operations complete even if an entire AZ plus an additional storage node are completely offline.
4. Storage nodes apply redo logs in parallel asynchronously, eliminating database crash recovery times and enabling sub-minute read replica additions with near-zero replication lag (<10ms).

---

# 5. Distributed Messaging & Observability (SQS, SNS, CloudWatch & X-Ray)

### 5.1 Definition & Core Concept
- **Amazon SQS (Simple Queue Service)**: Fully managed distributed message queuing service.
  - **Standard Queues**: Unlimited throughput, at-least-once delivery, best-effort ordering.
  - **FIFO Queues**: Up to 3,000 msg/sec with batching, exactly-once processing, strict ordering via Message Group ID.
- **Amazon SNS (Simple Notification Service)**: Fully managed pub/sub messaging engine. Decouples publishers from subscribers via topic fanout.
- **AWS CloudWatch & X-Ray**: Integrated observability stack delivering metric collection, threshold alarms, centralized log ingestion, and distributed trace visualization.

---

### 5.2 Internal Mechanics: Standard vs FIFO Queues, DLQ Redrive & X-Ray Trace Context

#### 1. SNS-to-SQS Fanout Architecture
A single message published to an SNS Topic is fanned out asynchronously to multiple independent SQS queues:

```
                          +-------------------+
                          |  Order Placed API |
                          +-------------------+
                                    │
                                    │ (Publish Event: "Order #9821")
                                    ▼
                         +---------------------+
                         |   SNS Topic (Orders)|
                         +---------------------+
                                    │
                ┌───────────────────┼───────────────────┐
                │                   │                   │
                ▼                   ▼                   ▼
      +-------------------+ +-------------------+ +-------------------+
      |  Billing SQS      | | Inventory SQS     | | Analytics SQS     |
      +-------------------+ +-------------------+ +-------------------+
                │                   │                   │
                ▼                   ▼                   ▼
      [Billing Worker]      [Inventory Worker]   [Analytics Ingest]
```

#### 2. Dead-Letter Queue (DLQ) & Visibility Timeout
When a consumer pulls a message from SQS via `ReceiveMessage`:
1. SQS starts the **Visibility Timeout** timer (default: 30s). The message is hidden from other workers.
2. If the worker processes the job successfully, it calls `DeleteMessage`.
3. If the worker crashes or errors, the visibility timeout expires. SQS makes the message visible again to other workers.
4. If a "poison pill" message fails repeatedly, `ReceiveCount` increments until it breaches `maxReceiveCount`, automatically shunting the message to a **Dead-Letter Queue (DLQ)** to prevent infinite loop poison pill thrashing.

---

### 5.3 Production Fanout Topology & Worker Implementation

```typescript
import { SQSClient, ReceiveMessageCommand, DeleteMessageCommand } from '@aws-sdk/client-sqs';

const sqs = new SQSClient({ region: 'us-east-1' });
const QUEUE_URL = process.env.ORDER_PROCESSING_QUEUE_URL!;

async function pollQueue(): Promise<void> {
  while (true) {
    try {
      const response = await sqs.send(
        new ReceiveMessageCommand({
          QueueUrl: QUEUE_URL,
          MaxNumberOfMessages: 10,
          WaitTimeSeconds: 20, // Long Polling to eliminate empty responses & reduce cost!
          VisibilityTimeout: 45,
        })
      );

      if (!response.Messages || response.Messages.length === 0) {
        continue;
      }

      for (const message of response.Messages) {
        try {
          await processOrder(JSON.parse(message.Body!));

          // Acknowledge and delete message upon successful processing
          await sqs.send(
            new DeleteMessageCommand({
              QueueUrl: QUEUE_URL,
              ReceiptHandle: message.ReceiptHandle!,
            })
          );
        } catch (processError) {
          console.error(`Failed processing message ${message.MessageId}. Letting visibility timeout expire:`, processError);
        }
      }
    } catch (pollError) {
      console.error('Error polling SQS:', pollError);
      await new Promise((resolve) => setTimeout(resolve, 5000));
    }
  }
}
```

---

### 5.4 Production Outages & Debugging

#### Outage Case Study: Short Polling SQS Bill Shock & Throttling
- **Context**: An enterprise microservice ran 20 background workers continuously polling an empty SQS queue with `WaitTimeSeconds: 0` (Short Polling).
- **Outage**: SQS API request counts spiked to 50,000,000 requests per day, resulting in a surprise $12,000 monthly AWS bill and client-side HTTP 429 throttling errors.
- **Resolution**: Enabled **Long Polling** by setting `WaitTimeSeconds: 20`. With long polling, SQS holds the connection open for up to 20 seconds waiting for messages before responding. Polling calls dropped by 99%, eliminating API costs and improving latency.

---

### 5.5 Trade-offs & Decision Matrix

| Queue Type | Throughput | Deduplication | Strict Ordering | Best Fit |
|:---|:---|:---|:---|:---|
| **SQS Standard** | Unlimited | None (At-least-once) | Best effort | Image processing, webhooks, asynchronous emails. |
| **SQS FIFO** | Up to 3,000/s (batch) | Exactly-once (Deduplication ID) | Guaranteed (Group ID) | Financial ledgers, stock inventory decrements, order sequencing. |
| **Amazon Kinesis** | Partition/Shard based | Custom offset tracking | Strict per-shard | Real-time clickstreams, log streaming, high-throughput metrics. |

---

### 5.6 Senior Interview Q&A

#### Q: How does AWS X-Ray propagate distributed trace headers across microservices, and how do you trace asynchronous SQS workflows?
**Answer**:
AWS X-Ray injects an HTTP header named `X-Amzn-Trace-Id` formatted as:
`Root=1-5e645f3e-1dfad07f0a06c6d0f620340d;Parent=0515810e74b5c73d;Sampled=1`.
1. **Synchronous HTTP Flow**: The reverse proxy or initial API gateway creates the `Root` trace ID. When the service makes an outbound HTTP request, the X-Ray SDK intercepts the call and injects the same `Root` ID into downstream request headers.
2. **Asynchronous SQS Flow**: HTTP headers do not carry over message queues natively. The publishing service must extract the current trace ID and inject it into the **SQS Message Attributes**:
   ```json
   "MessageAttributes": {
     "AWSTraceHeader": {
       "DataType": "String",
       "StringValue": "Root=1-5e645f3e...;Sampled=1"
     }
   }
   ```
3. When the consumer worker polls the message, the X-Ray SDK extracts `AWSTraceHeader`, sets it as the active segment's parent, and completes the distributed trace map across asynchronous boundaries.

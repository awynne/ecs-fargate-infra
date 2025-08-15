# Component Diagram - ECS Fargate Infrastructure

## Overview

This document provides detailed component diagrams showing the relationships and interactions between different parts of the ECS Fargate infrastructure.

## High-Level Component Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    AWS Account / Region                         │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                     VPC (10.0.0.0/16)                    │  │
│  │                                                           │  │
│  │  ┌─────────────────────────────────────────────────────┐  │  │
│  │  │              Public Subnet (10.0.1.0/24)           │  │  │
│  │  │                                                     │  │  │
│  │  │  ┌─────────────────┐    ┌─────────────────┐        │  │  │
│  │  │  │       ALB       │    │   ECS Fargate   │        │  │  │
│  │  │  │   (Port 80/443) │────│     Tasks       │        │  │  │
│  │  │  │                 │    │  (Port 8080)    │        │  │  │
│  │  │  └─────────────────┘    └─────────────────┘        │  │  │
│  │  │                                  │                 │  │  │
│  │  └──────────────────────────────────│─────────────────┘  │  │
│  │                                     │                    │  │
│  └─────────────────────────────────────│────────────────────┘  │
│                                        │                       │
│  ┌─────────────────────────────────────│────────────────────┐  │
│  │                                     ▼                    │  │
│  │            DynamoDB Tables (Managed Service)             │  │
│  │                                                          │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              Supporting Services                         │  │
│  │  ┌────────────┐ ┌────────────┐ ┌──────────────────────┐  │  │
│  │  │ CloudWatch │ │    ECR     │ │ Systems Manager      │  │  │
│  │  │    Logs    │ │ (Registry) │ │  Parameter Store     │  │  │
│  │  └────────────┘ └────────────┘ └──────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

## Detailed Component Breakdown

### Network Components

#### VPC Configuration
```
VPC: 10.0.0.0/16
├── Public Subnet: 10.0.1.0/24 (AZ: us-east-1a)
├── Internet Gateway
└── Route Table (0.0.0.0/0 → IGW)
```

#### Security Groups
```
ALB Security Group
├── Inbound: 0.0.0.0/0:80 (HTTP)
├── Inbound: 0.0.0.0/0:443 (HTTPS)
└── Outbound: ECS-SG:8080

ECS Security Group  
├── Inbound: ALB-SG:8080
└── Outbound: 0.0.0.0/0:443 (HTTPS for DynamoDB/AWS APIs)
```

### Compute Components

#### ECS Cluster Architecture
```
ECS Cluster: ecs-fargate-cluster
│
├── Service: web-service
│   ├── Task Definition: web-task:latest
│   │   ├── CPU: 256 units (0.25 vCPU)
│   │   ├── Memory: 512 MB
│   │   ├── Container: web-app
│   │   │   ├── Image: {account}.dkr.ecr.{region}.amazonaws.com/web-app:latest
│   │   │   ├── Port: 8080
│   │   │   └── Environment Variables (from Parameter Store)
│   │   └── IAM Task Role: ecs-task-role
│   │       ├── DynamoDB access
│   │       ├── Parameter Store access
│   │       └── CloudWatch Logs access
│   │
│   ├── Desired Count: 1 (development), 0 (off-hours)
│   ├── Auto Scaling Target: 1-3 tasks
│   └── Health Check: /health endpoint
│
└── Service Discovery: None (cost optimization)
```

#### Application Load Balancer
```
ALB: ecs-fargate-alb
│
├── Listener: HTTP:80
│   └── Action: Redirect to HTTPS
│
├── Listener: HTTPS:443
│   ├── SSL Certificate: AWS Certificate Manager
│   └── Target Group: ecs-tasks-tg
│       ├── Protocol: HTTP:8080
│       ├── Health Check: GET /health
│       ├── Health Check Interval: 30s
│       └── Healthy Threshold: 2
│
└── Access Logs: Disabled (cost optimization)
```

### Data Components

#### DynamoDB Design
```
DynamoDB Tables
│
├── Primary Table: app-data
│   ├── Partition Key: PK (String)
│   ├── Sort Key: SK (String) 
│   ├── Billing Mode: On-demand
│   ├── Encryption: AWS Managed Keys
│   └── Point-in-time Recovery: Disabled (cost)
│
└── Global Secondary Indexes: None initially
```

**Single Table Design Example**:
```
PK                    SK                    Data
USER#12345           PROFILE               {name, email, ...}
USER#12345           SESSION#abc123        {token, expires, ...}
PROJECT#67890        METADATA              {name, description, ...}
PROJECT#67890        TASK#001              {title, status, ...}
```

### Supporting Services

#### CloudWatch Configuration
```
CloudWatch
│
├── Log Groups
│   ├── /ecs/web-task (7-day retention)
│   ├── /aws/applicationloadbalancer/access (disabled)
│   └── Custom Application Logs
│
├── Metrics
│   ├── ECS Service Metrics (CPU, Memory)
│   ├── ALB Metrics (RequestCount, TargetResponseTime)
│   ├── DynamoDB Metrics (ConsumedCapacity)
│   └── Custom Application Metrics (limited to 10)
│
└── Alarms
    ├── ECS CPU > 80% (scale up)
    ├── ECS CPU < 20% (scale down)
    ├── ALB 5xx errors > 10/minute
    └── Monthly cost > $8 (80% of budget)
```

#### Systems Manager Parameter Store
```
Parameter Store
│
├── /app/environment (String)
├── /app/database/table-name (String)
├── /app/secrets/api-key (SecureString)
└── /app/config/log-level (String)
```

#### Elastic Container Registry
```
ECR Repository: web-app
├── URI: {account}.dkr.ecr.{region}.amazonaws.com/web-app
├── Lifecycle Policy: Keep last 10 images
└── Vulnerability Scanning: Enabled
```

## Data Flow Diagrams

### Request Processing Flow
```
Internet User
     │ HTTPS (443)
     ▼
Application Load Balancer
     │ HTTP (8080)
     ▼
ECS Fargate Task
     │ AWS SDK
     ▼
DynamoDB Table
     │ Response
     ▼
ECS Fargate Task
     │ HTTP Response
     ▼
Application Load Balancer
     │ HTTPS Response
     ▼
Internet User
```

### Deployment Flow
```
Developer
     │ docker push
     ▼
ECR Repository
     │ new image
     ▼
ECS Service Update
     │ rolling deployment
     ▼
New Task Definition
     │ health check
     ▼
Target Group Registration
     │ traffic routing
     ▼
Active Application
```

### Auto Scaling Flow
```
CloudWatch Metrics
     │ CPU > 80%
     ▼
Auto Scaling Policy
     │ scale out
     ▼
ECS Service
     │ launch new task
     ▼
Task Placement
     │ register with ALB
     ▼
Load Distribution
```

## Security Architecture

### IAM Roles and Policies
```
ECS Task Execution Role
├── AmazonECSTaskExecutionRolePolicy
├── ECR access (pull images)
└── CloudWatch Logs access

ECS Task Role
├── DynamoDB access (specific tables)
├── Parameter Store read access
└── CloudWatch custom metrics

ALB Service Role
└── (Managed by AWS)
```

### Network Security
```
Defense in Depth
│
├── Edge: ALB with WAF (optional, cost consideration)
├── Network: Security Groups (stateful firewall)
├── Application: Container-level security
└── Data: DynamoDB encryption at rest
```

## Cost Optimization Components

### Free Tier Usage Tracking
```
Cost Monitoring
│
├── ECS Fargate: 20GB-hours/month
├── DynamoDB: 25GB + 25 RCU/WCU
├── ALB: 750 hours + 15GB processing
├── CloudWatch: 5GB logs + 10 metrics
└── Parameter Store: 10,000 parameters
```

### Resource Scheduling
```
Auto Scaling Schedule
│
├── Business Hours (9 AM - 6 PM): Min 1, Max 3 tasks
├── Off Hours (6 PM - 9 AM): Min 0, Max 1 tasks  
└── Weekends: Min 0, Max 0 tasks
```

This component architecture provides a comprehensive view of how all pieces fit together while maintaining cost optimization and Free Tier compliance.
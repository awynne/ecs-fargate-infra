# ECS Fargate Infrastructure - System Overview

## Architecture Summary

The ECS Fargate infrastructure implements a modern containerized architecture optimized for AWS Free Tier usage, targeting demo and development workloads with cost controls and scalability.

## High-Level Architecture

```
Internet
    │
    ▼
┌─────────────────┐
│ Application     │ ── ALB (Application Load Balancer)
│ Load Balancer   │    - SSL Termination
└─────────────────┘    - Health Checks
    │                  - 750 hours/month Free Tier
    ▼
┌─────────────────┐
│ ECS Fargate     │ ── Container Orchestration
│ Cluster         │    - 0.25 vCPU, 512MB memory
└─────────────────┘    - Auto-scaling to zero
    │                  - 20GB-hours Free Tier
    ▼
┌─────────────────┐
│ DynamoDB        │ ── NoSQL Database
│ Tables          │    - Always Free: 25GB + 25 RCU/WCU
└─────────────────┘    - Single-table design
    │
    ▼
┌─────────────────┐
│ CloudWatch      │ ── Logging & Monitoring
│ Logs            │    - 7-day retention
└─────────────────┘    - 5GB ingestion Free Tier
```

## Core Components

### 1. Virtual Private Cloud (VPC)
- **Purpose**: Network isolation and security
- **Configuration**: Single AZ deployment to minimize data transfer costs
- **Subnets**: Public subnets only (no NAT Gateway to avoid $45/month cost)
- **Security**: Security groups for network-level access control

### 2. Application Load Balancer (ALB)
- **Purpose**: Traffic distribution and SSL termination
- **Features**: 
  - Health checks for ECS tasks
  - SSL/TLS termination
  - Request routing
- **Free Tier**: 750 hours/month + 15GB data processing

### 3. ECS Fargate Cluster
- **Purpose**: Container orchestration without server management
- **Task Specifications**:
  - CPU: 0.25 vCPU (minimum Fargate allocation)
  - Memory: 512MB (minimum Fargate allocation)
  - Container image: Application containers
- **Auto-scaling**: Scale to zero during off-hours
- **Free Tier**: 20GB-hours storage + 5GB ephemeral storage/month

### 4. DynamoDB Database
- **Purpose**: Persistent data storage
- **Advantages**: Always Free tier (25GB storage + 25 RCU/WCU)
- **Design Pattern**: Single-table design for cost optimization
- **Backup**: Point-in-time recovery (additional cost consideration)

### 5. CloudWatch Logging
- **Purpose**: Application and infrastructure monitoring
- **Configuration**: 7-day log retention to stay within Free Tier
- **Free Tier**: 5GB log ingestion + 10 custom metrics

### 6. Systems Manager Parameter Store
- **Purpose**: Configuration and secrets management
- **Free Tier**: 10,000 parameters included
- **Security**: SecureString parameters for sensitive data

## Network Architecture

### Single Availability Zone Design
```
VPC (10.0.0.0/16)
│
├── Public Subnet (10.0.1.0/24)
│   ├── Application Load Balancer
│   ├── ECS Fargate Tasks
│   └── NAT Instance (if needed, t2.micro)
│
└── Security Groups
    ├── ALB Security Group (80, 443)
    ├── ECS Security Group (8080)
    └── Database Security Group (DynamoDB - managed)
```

**Note**: Single AZ deployment minimizes cross-AZ data transfer charges while sacrificing high availability for cost optimization.

## Data Flow

### Request Flow
1. **User Request** → Application Load Balancer
2. **ALB** → ECS Fargate Task (health check + route)
3. **ECS Task** → DynamoDB (data operations)
4. **DynamoDB** → ECS Task (response data)
5. **ECS Task** → ALB → User (HTTP response)

### Logging Flow
1. **Application Logs** → CloudWatch Log Groups
2. **ECS Logs** → CloudWatch Container Insights
3. **ALB Logs** → CloudWatch (optional, costs extra)

## Security Model

### Network Security
- VPC with private IP addressing
- Security groups with least-privilege access
- No direct internet access to ECS tasks (through ALB only)

### Data Security
- DynamoDB encryption at rest (managed by AWS)
- SSL/TLS encryption in transit via ALB
- IAM roles for service-to-service authentication

### Secrets Management
- Application secrets via Systems Manager Parameter Store
- IAM roles instead of access keys
- No hardcoded credentials in container images

## Scalability Design

### Horizontal Scaling
- ECS Auto Scaling based on CPU/memory utilization
- DynamoDB on-demand billing for automatic scaling
- ALB distributes traffic across multiple ECS tasks

### Cost-Optimized Scaling
- Scale to zero during off-hours (e.g., nights/weekends)
- DynamoDB provisioned capacity for predictable workloads
- CloudWatch alarms for cost threshold monitoring

## Monitoring & Observability

### Key Metrics
- ECS task CPU/memory utilization
- DynamoDB read/write capacity utilization
- ALB response times and error rates
- Free Tier usage across all services

### Alerting
- Cost threshold alerts (80% of $10 budget)
- Service health alerts (task failures, high error rates)
- Free Tier usage warnings

## Free Tier Optimization

### Monthly Limits
- **ECS Fargate**: 20GB-hours storage + 5GB ephemeral
- **DynamoDB**: 25GB storage + 25 RCU/WCU (always free)
- **ALB**: 750 hours + 15GB data processing
- **CloudWatch**: 5GB log ingestion + 10 custom metrics
- **Parameter Store**: 10,000 parameters

### Cost Controls
1. Single AZ deployment
2. Minimal resource allocations
3. Auto-scaling to zero
4. 7-day log retention
5. On-demand DynamoDB billing
6. Resource tagging for cost attribution

## Alternative Configurations

### High Availability Option
For production use (outside Free Tier scope):
- Multi-AZ deployment
- RDS with Multi-AZ for database
- Auto Scaling across multiple AZs
- Enhanced monitoring and logging

### Serverless Alternative
Alternative architecture using:
- Lambda functions instead of ECS
- API Gateway instead of ALB
- DynamoDB (same)
- CloudWatch (same)

## Implementation Phases

### Phase 1: Core Infrastructure
- VPC and networking
- ECS cluster setup
- Basic ALB configuration

### Phase 2: Application Deployment
- Container registry (ECR)
- ECS task definitions
- Application deployment

### Phase 3: Data Layer
- DynamoDB table setup
- Application integration
- Data migration tools

### Phase 4: Monitoring & Operations
- CloudWatch dashboards
- Cost monitoring
- Operational runbooks
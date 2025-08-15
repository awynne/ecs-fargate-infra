# ECS Fargate Infrastructure - System Overview

## Architecture Summary

The ECS Fargate infrastructure implements a modern containerized architecture optimized for **Ruby on Rails applications** with **PostgreSQL**, targeting demo and development workloads with AWS Free Tier cost controls and scalability.

## High-Level Architecture

```
Internet
    │
    ▼
┌─────────────────┐
│ Application     │ ── ALB (Application Load Balancer)
│ Load Balancer   │    - SSL Termination & Health Checks
└─────────────────┘    - 750 hours/month Free Tier
    │
    ▼
┌─────────────────┐
│ ECS Fargate     │ ── Rails Application Containers
│ Cluster         │    - 0.25 vCPU, 512MB memory
└─────────────────┘    - Auto-scaling capabilities
    │                  - 20GB-hours Free Tier
    ▼
┌─────────────────┐
│ RDS PostgreSQL  │ ── Relational Database
│ (db.t3.micro)   │    - 750 hours/month Free Tier
└─────────────────┘    - 20GB storage + 20GB backup
    │
    ▼
┌─────────────────┐
│ CloudWatch      │ ── Logging & Monitoring
│ Logs            │    - 7-day retention
└─────────────────┘    - 5GB ingestion Free Tier
```

## Rails Application Benefits

- ✅ **Full ActiveRecord ORM**: Native PostgreSQL support with migrations, associations, validations
- ✅ **Rails Ecosystem**: Standard gems, generators, and development workflow maintained
- ✅ **Database Features**: ACID compliance, complex queries, joins, and transactions
- ✅ **Development Tools**: Rails console, rake tasks, and familiar database tooling

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

### 4. RDS PostgreSQL Database
- **Purpose**: Relational data storage for Rails ActiveRecord ORM
- **Instance**: db.t3.micro (Free Tier eligible - 750 hours/month)
- **Storage**: 20GB General Purpose SSD (included in Free Tier)
- **Features**: Automated backups, point-in-time recovery, maintenance windows
- **Cost Optimization**: Scheduled start/stop during off-hours for development use

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
1. **User Request** → Application Load Balancer (HTTPS)
2. **ALB** → ECS Fargate Task running Rails application (health check + route)
3. **Rails App** → RDS PostgreSQL (ActiveRecord database operations)
4. **PostgreSQL** → Rails App (query results)
5. **Rails App** → ALB → User (HTTP response)

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
- RDS PostgreSQL encryption at rest (managed by AWS)
- SSL/TLS encryption in transit via ALB and RDS connections
- IAM roles for service-to-service authentication
- Database credentials managed via Systems Manager Parameter Store

### Secrets Management
- Application secrets via Systems Manager Parameter Store
- IAM roles instead of access keys
- No hardcoded credentials in container images

## Scalability Design

### Horizontal Scaling
- ECS Auto Scaling based on CPU/memory utilization
- ALB distributes traffic across multiple Rails application tasks
- RDS PostgreSQL connection pooling for efficient database connections

### Cost-Optimized Scaling
- ECS tasks scale to zero during off-hours (e.g., nights/weekends)
- RDS scheduled start/stop automation for development hours only
- CloudWatch alarms for cost threshold monitoring
- Rails application optimized for quick startup times

## Monitoring & Observability

### Key Metrics
- ECS task CPU/memory utilization for Rails applications
- RDS PostgreSQL CPU/memory utilization and connection count
- ALB response times and error rates
- Rails application response times and database query performance
- Free Tier usage across ECS and RDS services

### Alerting
- Cost threshold alerts (80% of $10 budget)
- RDS Free Tier hour consumption warnings (600 hours = 80% threshold)
- Service health alerts (task failures, high error rates, database connection issues)
- Rails application error rate monitoring

## Free Tier Optimization

### Monthly Limits
- **ECS Fargate**: 20GB-hours storage + 5GB ephemeral storage
- **RDS PostgreSQL**: 750 hours db.t3.micro + 20GB storage + 20GB backup
- **ALB**: 750 hours + 15GB data processing
- **CloudWatch**: 5GB log ingestion + 10 custom metrics
- **Parameter Store**: 10,000 parameters

### Cost Controls
1. Single AZ deployment (no cross-AZ data transfer charges)
2. Minimal resource allocations (0.25 vCPU, 512MB memory)
3. ECS auto-scaling to zero during off-hours
4. RDS scheduled start/stop automation for development hours
5. 7-day CloudWatch log retention
6. Resource tagging for cost attribution and monitoring

## Alternative Configurations

### High Availability Option
For production use (outside Free Tier scope):
- Multi-AZ deployment
- RDS with Multi-AZ for database
- Auto Scaling across multiple AZs
- Enhanced monitoring and logging

### Serverless Alternative
Alternative architecture for Rails using:
- Lambda functions with Jets.rb framework instead of ECS
- API Gateway instead of ALB  
- Aurora Serverless PostgreSQL instead of RDS
- CloudWatch (same monitoring and logging)

## Implementation Phases

### Phase 1: Core Infrastructure
- VPC and networking
- ECS cluster setup
- Basic ALB configuration

### Phase 2: Application Deployment
- Container registry (ECR) for Rails application images
- ECS task definitions with Rails-specific configuration
- Rails application deployment with health checks

### Phase 3: Data Layer
- RDS PostgreSQL instance setup
- Database security groups and parameter configuration
- Rails database configuration and connection pooling
- Database migrations and seed data

### Phase 4: Monitoring & Operations
- CloudWatch dashboards
- Cost monitoring
- Operational runbooks
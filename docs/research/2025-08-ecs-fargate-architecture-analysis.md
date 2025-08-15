# ECS Fargate Infrastructure Architecture Analysis

**Date**: 2025-08-15  
**Author**: Architecture Research  
**JIRA**: WS-20  
**Status**: In Progress  

## Objective

Research and propose an architecture design for the ECS Fargate infrastructure, comparing alternative approaches and making recommendations optimized for AWS Free Tier usage.

## Current State Analysis

### Repository State
- **Empty implementation**: No Terraform code, scripts, or Docker files currently exist
- **Documentation structure**: Recently established docs/ structure with comprehensive organization
- **Project scope**: Terraform-managed AWS ECS Fargate infrastructure for demo/development workloads
- **Constraints**: AWS Free Tier optimization with cost controls

### Planned Components (from CLAUDE.md)
- **VPC**: Single AZ custom VPC (no NAT Gateway)
- **ECS**: Fargate tasks with 0.25 vCPU, 512MB memory
- **Load Balancer**: Application Load Balancer
- **Database**: RDS db.t3.micro PostgreSQL or DynamoDB alternative
- **Logging**: CloudWatch with 7-day retention
- **Secrets**: SSM Parameter Store

## Architecture Approaches Analysis

### Approach 1: Basic ECS Fargate with RDS
**Components**:
- VPC with public subnets only (no NAT Gateway)
- ECS Fargate cluster with minimal task specs
- Application Load Balancer
- RDS PostgreSQL db.t3.micro
- CloudWatch logging

**Pros**:
- Traditional 3-tier architecture pattern
- PostgreSQL provides ACID compliance
- ECS Fargate eliminates server management
- ALB provides health checks and SSL termination

**Cons**:
- RDS consumes significant Free Tier hours (750/month)
- Database always running incurs costs
- Limited to single AZ for cost optimization
- Backup storage counts against Free Tier limits

**Free Tier Impact**:
- RDS: 750 hours db.t3.micro + 20GB storage
- ECS Fargate: 20GB-hours + 5GB ephemeral storage
- ALB: 750 hours + 15GB data processing
- CloudWatch: 5GB log ingestion

### Approach 2: Serverless with DynamoDB
**Components**:
- Lambda functions for compute
- API Gateway for HTTP interface
- DynamoDB for persistence
- CloudWatch for logging
- Optional CloudFront for static assets

**Pros**:
- True serverless with pay-per-use
- DynamoDB has generous always-free tier (25GB, 25 RCU/WCU)
- Lambda scales to zero automatically
- No infrastructure management required
- Better cost efficiency for sporadic usage

**Cons**:
- Different programming model (event-driven)
- Cold start latency for Lambda
- DynamoDB NoSQL limitations for complex queries
- Vendor lock-in to AWS services

**Free Tier Impact**:
- Lambda: 1M requests + 400K GB-seconds/month
- DynamoDB: 25GB storage + 25 RCU/WCU (always free)
- API Gateway: 1M requests/month
- CloudWatch: 5GB log ingestion

### Approach 3: Hybrid ECS Fargate with DynamoDB
**Components**:
- ECS Fargate for containerized applications
- DynamoDB for data persistence
- Application Load Balancer
- VPC with public subnets
- CloudWatch logging

**Pros**:
- Container-based deployment flexibility
- DynamoDB's always-free tier reduces costs
- Familiar container orchestration
- Can scale tasks to zero during off-hours
- Lower operational complexity than RDS

**Cons**:
- NoSQL data modeling complexity
- Limited to DynamoDB query patterns
- ECS tasks must handle DynamoDB SDK integration
- Still requires ALB running costs

**Free Tier Impact**:
- ECS Fargate: 20GB-hours + 5GB ephemeral storage
- DynamoDB: 25GB + 25 RCU/WCU (always free)
- ALB: 750 hours + 15GB data processing
- CloudWatch: 5GB log ingestion

### Approach 4: Single EC2 Instance Alternative
**Components**:
- Single t2.micro EC2 instance
- Docker Compose for container orchestration
- Local PostgreSQL or SQLite
- Nginx for load balancing/SSL termination
- CloudWatch agent for logging

**Pros**:
- Maximum Free Tier utilization (750 hours t2.micro)
- Full control over the stack
- Can run multiple services on single instance
- Lower data transfer costs (single AZ)
- Simple architecture

**Cons**:
- Single point of failure
- Manual scaling and management
- No built-in high availability
- Requires system administration
- Less cloud-native approach

**Free Tier Impact**:
- EC2: 750 hours t2.micro + 30GB EBS storage
- CloudWatch: 5GB log ingestion
- Data transfer: 15GB outbound

## Recommendations Analysis

### Primary Recommendation: Hybrid ECS Fargate with DynamoDB (Approach 3)

**Rationale**:
1. **Cost Optimization**: DynamoDB's always-free tier (25GB + 25 RCU/WCU) provides persistent storage without consuming RDS hours
2. **Container Benefits**: Maintains container-based deployment while optimizing costs
3. **Scalability**: ECS tasks can scale to zero during off-hours
4. **Future Growth**: Architecture can evolve with application needs
5. **Cloud-Native**: Leverages managed services appropriately

**Implementation Strategy**:
1. Start with minimal ECS task configuration (0.25 vCPU, 512MB memory)
2. Use DynamoDB single-table design for initial simplicity
3. Implement auto-scaling policies for cost control
4. Use CloudWatch for monitoring and alerting

### Secondary Recommendation: Serverless with DynamoDB (Approach 2)

**Use Case**: If the application is event-driven or has sporadic usage patterns

**Benefits**:
- Optimal cost efficiency for low-traffic scenarios
- Automatic scaling and infrastructure management
- Faster initial deployment

## Next Steps

1. **Architecture Decision**: Create ADR documenting the chosen approach
2. **Technical Design**: Develop detailed architecture diagrams and component specifications
3. **Implementation Plan**: Create phased implementation approach
4. **Cost Monitoring**: Design monitoring and alerting for Free Tier usage

## Research Conclusions

The hybrid ECS Fargate with DynamoDB approach provides the best balance of:
- Free Tier cost optimization
- Container-based deployment benefits
- Managed service advantages
- Future scalability options

This approach aligns with the project's demo/development focus while maintaining cloud-native principles and cost controls.
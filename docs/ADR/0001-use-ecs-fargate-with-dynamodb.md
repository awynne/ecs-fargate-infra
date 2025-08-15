# ADR-0001: Use ECS Fargate with RDS PostgreSQL for Ruby on Rails Infrastructure

## Status
Accepted

## Context

We need to design a cloud infrastructure architecture for the ecs-fargate-infra project that meets the following requirements:

- **Application Constraint**: Must support an existing **Ruby on Rails application** using **PostgreSQL**
- **Cost Optimization**: Must operate within AWS Free Tier limits for demo/development workloads
- **Container Support**: Should leverage containerized application deployment
- **Managed Services**: Prefer AWS managed services to reduce operational overhead
- **Development Focus**: Optimized for development and demonstration use cases
- **Rails Compatibility**: Must maintain ActiveRecord ORM and Rails ecosystem benefits

Several architectural approaches were evaluated:

1. **ECS Fargate with RDS PostgreSQL**: Traditional 3-tier architecture ✅ **Selected**
2. **Serverless with Lambda + DynamoDB**: Event-driven serverless approach
3. **ECS Fargate with DynamoDB**: Hybrid container + NoSQL approach  
4. **Single EC2 instance**: Simple monolithic deployment

**Critical Discovery**: The application is a Ruby on Rails app with PostgreSQL dependency, which eliminates DynamoDB options due to ActiveRecord ORM incompatibility.

## Decision

We will implement **ECS Fargate with RDS PostgreSQL** as the primary infrastructure architecture.

### Core Components:
- **Container Orchestration**: Amazon ECS with Fargate launch type
- **Database**: Amazon RDS PostgreSQL with Free Tier optimization
- **Load Balancer**: Application Load Balancer (ALB)
- **Networking**: VPC with single AZ deployment
- **Logging**: CloudWatch with 7-day retention
- **Secrets**: Systems Manager Parameter Store

### Resource Specifications:
- **ECS Tasks**: 0.25 vCPU, 512MB memory (minimum Fargate allocation)
- **RDS PostgreSQL**: db.t3.micro instance with 20GB storage
- **ALB**: Single load balancer with SSL termination
- **VPC**: Single AZ with public subnets (no NAT Gateway)

## Consequences

### Positive Consequences

**Rails Application Benefits**:
- ✅ **Full ActiveRecord ORM Support**: No application code changes required
- ✅ **Rails Ecosystem Compatibility**: Migrations, generators, and Rails patterns maintained
- ✅ **PostgreSQL Features**: ACID compliance, complex queries, joins, and transactions
- ✅ **Development Workflow**: Standard Rails console, rake tasks, and database tooling
- ✅ **Team Productivity**: No learning curve for NoSQL data modeling

**Cost Optimization**:
- RDS Free Tier: 750 hours/month db.t3.micro sufficient for development workloads
- ECS Fargate: 20GB-hours monthly allocation for containerized Rails application
- Single AZ deployment eliminates cross-AZ data transfer costs
- Scheduled scaling: Stop RDS during off-hours (nights/weekends) for additional cost savings
- 20GB storage + 20GB backup included in Free Tier

**Technical Benefits**:
- Container-based Rails deployment with familiar workflow
- Fargate eliminates server management overhead
- RDS provides automated backups and point-in-time recovery
- ALB provides health checks and SSL termination for Rails application

**Operational Advantages**:
- All managed services reduce operational complexity
- CloudWatch integration provides comprehensive monitoring
- Infrastructure as Code (Terraform) enables reproducible deployments
- Auto-scaling capabilities for Rails application workloads

### Negative Consequences

**Cost Considerations**:
- RDS consumes time-limited Free Tier hours (750/month vs DynamoDB always-free)
- Database must run continuously or require scheduled start/stop automation
- Backup storage counts against 20GB Free Tier limit
- Single AZ deployment sacrifices high availability for cost optimization

**Technical Limitations**:
- Minimum Fargate resource allocation (0.25 vCPU, 512MB) may be over-provisioned
- RDS db.t3.micro has limited performance compared to larger instances
- Single AZ deployment creates potential single point of failure
- Limited to 20GB storage in Free Tier (can auto-scale but incurs costs)

**Operational Considerations**:
- RDS requires monitoring for Free Tier hour consumption
- Database connection pooling needed for Rails application efficiency
- Container images must include PostgreSQL client libraries and gems
- Backup management within Free Tier storage limits

### Migration Path

If requirements change, the architecture supports evolution:

**To High Availability**: 
- Expand to multi-AZ RDS deployment (additional cost)
- Deploy ECS tasks across multiple AZs
- Implement cross-region disaster recovery

**To Production Scale**:
- Upgrade to larger RDS instance classes (db.t3.small, db.t3.medium)
- Increase ECS task resources and auto-scaling limits
- Add RDS read replicas for read scaling

**To Serverless Alternative**:
- Migrate Rails application to Lambda using frameworks like Jets.rb
- Replace ALB with API Gateway
- Migrate PostgreSQL to Aurora Serverless for pay-per-request billing

### Monitoring and Validation

**Success Metrics**:
- Monthly AWS costs remain under $10 (Free Tier + small buffer)
- Rails application deployment time < 10 minutes
- Database migrations execute successfully in containerized environment
- Rails application responds within 500ms for typical requests
- RDS Free Tier hours consumed < 600/month (80% threshold)

**Risk Mitigation**:
- RDS scheduling automation to stop database during off-hours
- Daily cost monitoring with CloudWatch
- Free Tier usage tracking and alerting for both ECS and RDS
- Rails application health checks and database connection monitoring
- Cost threshold alerts at 80% of budget ($8/month)

## References

- [AWS Free Tier Details](https://aws.amazon.com/free/)
- [ECS Fargate Pricing](https://aws.amazon.com/fargate/pricing/)
- [DynamoDB Free Tier](https://aws.amazon.com/dynamodb/pricing/)
- Research Document: [2025-08-ecs-fargate-architecture-analysis.md](../research/2025-08-ecs-fargate-architecture-analysis.md)
- JIRA Ticket: WS-20
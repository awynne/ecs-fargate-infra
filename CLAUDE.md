# AWS ECS Fargate Infrastructure

## Overview
Terraform-managed AWS ECS Fargate infrastructure configured for Free Tier usage. Targets demo and development workloads with cost optimization.

## Project Structure
```
.
├── terraform/                 # Infrastructure as code
│   ├── environments/dev/     # Single environment (Free Tier)
│   ├── modules/              # Reusable modules
│   │   ├── ecs-cluster/
│   │   ├── ecs-service/
│   │   ├── vpc/
│   │   └── alb/
│   └── global/               # IAM roles, state bucket
├── scripts/                  # Automation scripts
│   ├── deploy.py
│   ├── parameter-store.py
│   ├── check-free-tier.py
│   └── cost-monitor.py
├── docs/                     # Technical documentation
│   ├── architecture/
│   ├── runbooks/
│   └── decisions/            # ADRs
├── docker/                   # Container definitions
└── .github/workflows/        # CI/CD automation
```

## Documentation Strategy

### Documentation Types and Locations
- **Technical specs**: `docs/` directory in source control
- **Task management**: JIRA tickets for status tracking
- **Research/spikes**: Confluence wiki pages for evaluation
- **Architecture decisions**: `docs/decisions/` as ADR files
- **Operational procedures**: `docs/runbooks/`

### Documentation Workflow
1. **Research phase**: Document findings in Confluence "Software Development" space under `ecs-fargate-infra` page tree
2. **Design phase**: Create technical docs in `docs/architecture/`
3. **Implementation**: Update docs during development
4. **Operations**: Maintain runbooks in `docs/runbooks/`

### Tool Configuration
- **GitHub**: Use `gh` CLI with `awynne` account for repository operations
- **JIRA**: Use MCP server integration with WebStore project for task management
- **Confluence**: Use Atlassian MCP with "Software Development" space, repo-specific page tree: `ecs-fargate-infra`
- **Documentation**: Use context7 MCP for language syntax and framework documentation

## Prerequisites
- AWS Free Tier account (12-month eligible or Always Free services)
- AWS CLI configured with appropriate credentials
- Terraform >= 1.5
- Docker
- Python 3.9+ or Ruby 2.7+
- AWS Account with appropriate IAM permissions

## AWS Free Tier Optimization

### Free Tier Limits & Usage
- **EC2**: 750 hours/month of t2.micro instances (covered by Fargate free tier)
- **Fargate**: 20 GB-hours storage + 5 GB ephemeral storage per month
- **ALB**: 750 hours/month + 15 GB data processing
- **RDS**: 750 hours of db.t3.micro + 20GB storage + 20GB backup
- **CloudWatch**: 10 custom metrics, 1M API requests, 5GB log ingestion
- **S3**: 5GB storage, 20K GET requests, 2K PUT requests
- **Lambda**: 1M requests + 400K GB-seconds compute time/month

### Cost Controls
- Single dev environment only
- Auto-scaling with scale-to-zero during off-hours
- Single AZ deployment to avoid cross-AZ data transfer
- 7-day CloudWatch log retention
- Resource tagging for cost attribution

## Environment Setup

### AWS Credentials
Use AWS IAM roles or AWS SSO for authentication. Never commit AWS access keys.

```bash
# Configure AWS CLI
aws configure sso

# Or use environment variables (for CI/CD)
export AWS_ROLE_ARN="arn:aws:iam::ACCOUNT:role/DeploymentRole"
export AWS_WEB_IDENTITY_TOKEN_FILE=/var/run/secrets/eks.amazonaws.com/serviceaccount/token
```

## Terraform Best Practices

### State Management
- Use remote state with S3 backend and DynamoDB locking
- Separate state files per environment
- Enable versioning and encryption on S3 bucket

### Secrets Management
**NEVER commit secrets to source control**

#### Using AWS Secrets Manager
```hcl
# Reference secrets in Terraform
data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = "rds-password-${var.environment}"
}

locals {
  db_password = jsondecode(data.aws_secretsmanager_secret_version.db_password.secret_string)["password"]
}
```

#### Using AWS Systems Manager Parameter Store
```hcl
# For non-sensitive configuration
data "aws_ssm_parameter" "app_config" {
  name = "/app/${var.environment}/config"
}
```

#### Environment Variables for Terraform
```bash
# Use .env files for local development (never commit)
export TF_VAR_db_username="admin"
export TF_VAR_environment="dev"
```

## Infrastructure Components
- **VPC**: Single AZ custom VPC (no NAT Gateway)
- **ECS**: Fargate tasks with 0.25 vCPU, 512MB memory
- **Load Balancer**: Application Load Balancer
- **Database**: 
  - RDS db.t3.micro PostgreSQL (750 hours/month limit)
  - DynamoDB alternative (25GB, 25 RCU/WCU always free)
- **Logging**: CloudWatch with 7-day retention
- **Secrets**: SSM Parameter Store (10K parameters free)

### Free Tier Resource Specifications
```hcl
# ECS Task Definition - Free Tier Optimized
resource "aws_ecs_task_definition" "app" {
  family                   = "app-task"
  requires_attributes      = []
  network_mode            = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                     = "256"  # 0.25 vCPU (minimum)
  memory                  = "512"  # 512 MB (minimum)
  
  container_definitions = jsonencode([{
    name  = "app"
    image = "nginx:alpine"  # Lightweight base image
    # ... minimal configuration
  }])
}

# RDS Instance - Free Tier
resource "aws_db_instance" "main" {
  identifier             = "demo-db"
  engine                = "postgres"
  engine_version        = "13.13"
  instance_class        = "db.t3.micro"  # Free Tier eligible
  allocated_storage     = 20             # Free Tier limit
  max_allocated_storage = 20             # Prevent auto-scaling costs
  storage_encrypted     = false          # Encryption costs extra
  
  backup_retention_period = 7            # Minimum for point-in-time recovery
  backup_window          = "03:00-04:00" # Low-traffic hours
  maintenance_window     = "sun:04:00-sun:05:00"
  
  skip_final_snapshot = true             # Avoid snapshot costs for demo
}
```

### Security Configuration
- TLS encryption in transit via ALB
- IAM roles with least privilege access
- Security groups restrict access to required ports
- SSM Parameter Store SecureString for sensitive values
- **Excluded services** (cost implications):
  - VPC Flow Logs (CloudWatch ingestion costs)
  - GuardDuty ($1.40+/month after trial)
  - Config (per configuration item charges)
  - RDS encryption at rest (additional storage costs)

## Development Workflow

### Setup
```bash
git clone <repository-url>
cd ecs-fargate-infra
./scripts/setup-local-env.py
cd terraform/environments/dev
terraform init
```

### Deployment
```bash
./scripts/deploy.py --environment dev --tier free
./scripts/check-free-tier-usage.py --service all
```

### Parameter Management
```bash
./scripts/parameter-store.py create --name "/app/dev/db-password" --type SecureString
./scripts/parameter-store.py create --name "/app/dev/config" --from-file config.json
```

## CI/CD Pipeline

### CI/CD Pipeline
- **Pull requests**: Terraform validation, plan, security scan
- **Main branch**: Deploy to dev environment
- **Manual trigger**: Production deployment (if needed)

### Pipeline stages
1. `terraform validate` and `terraform fmt -check`
2. `tfsec` security scanning
3. `terraform plan` 
4. `terraform apply` on merge
5. Health check validation

## Monitoring

### CloudWatch Setup
- ECS service metrics (included)
- Critical alarms only (10 alarm limit)
- 7-day log retention (5GB/month limit)
- ALB health checks

### Cost Monitoring  
```bash
./scripts/setup-billing-alerts.py --budget-limit 10
./scripts/check-daily-costs.py --services ec2,rds,ecs,cloudwatch
```

## Helper Scripts

#### deploy.py
- Pre-deployment Free Tier usage validation
- Terraform deployment automation
- Cost monitoring integration
- Failed deployment cleanup

#### parameter-store.py  
- SSM Parameter Store management
- Secure parameter creation and updates
- Environment-specific configurations

#### check-free-tier-usage.py
- AWS service usage tracking
- Free Tier limit monitoring
- Monthly usage reporting
- Overage prevention alerts

#### cost-monitor.py
- Daily cost tracking
- Budget threshold alerts  
- Resource usage analysis

## Security

### Code Security
- No hardcoded secrets or credentials
- SSM Parameter Store for sensitive values
- GitHub security advisories for dependency scanning
- Signed commits required

### Infrastructure Security  
- Resource tagging for governance
- TLS encryption in transit via ALB
- Security group network segmentation
- IAM roles with least privilege

### Access Control
- IAM roles instead of access keys
- MFA on AWS root account  
- CloudTrail event history (90 days included)

## Common Commands

### Terraform Commands
```bash
terraform fmt -recursive .
terraform validate
tfsec .
terraform plan -var-file="environments/dev/terraform.tfvars"
terraform apply -var-file="environments/dev/terraform.tfvars"
```

### Container Commands
```bash
docker build -t app:latest .
./scripts/push-to-ecr.py --tag latest --environment dev
```

### GitHub Operations
```bash
# Create pull request
gh pr create --title "Feature: Add ECS service" --body "Implementation details"

# Check PR status
gh pr status

# Merge PR
gh pr merge --squash
```

### JIRA Integration
```bash
# Create task (via MCP server integration)
# Example: Create WebStore project task for infrastructure changes
# Tasks created through configured MCP integration with WebStore project

# Update task status
# Status updates managed through JIRA WebStore project board
```

### ECS Commands
```bash
./scripts/ecs-deploy.py --service app-service --environment dev
./scripts/ecs-logs.py --service app-service --environment dev --follow
./scripts/ecs-scale.py --service app-service --desired-count 0
```

### Cost Management
```bash
./scripts/check-free-tier-usage.py --detailed
./scripts/estimate-costs.py --environment dev  
./scripts/cleanup-unused-resources.py --dry-run
```

## Troubleshooting

#### Common Issues
1. **State lock**: `terraform force-unlock <lock-id>`
2. **Task failures**: Check CloudWatch logs  
3. **Health check failures**: Verify security groups
4. **RDS connection**: Check VPC routing and security groups
5. **Free Tier exceeded**: Monitor with `./scripts/check-free-tier-usage.py`
6. **Unexpected charges**: Review AWS Billing Dashboard

#### Debug Commands
```bash
aws ecs describe-services --cluster <cluster-name> --services <service-name>
aws logs tail /ecs/<service-name> --follow  
aws elbv2 describe-target-health --target-group-arn <target-group-arn>
```

## Development Workflow

### JIRA-Driven Development
**Requirement**: All work must have a corresponding JIRA ticket in the WebStore project
- No development work without a defined ticket
- Ticket must contain acceptance criteria and task definition
- If work is requested without a ticket, push back and require ticket creation first

### Git Branching Strategy
```bash
# Create feature branch from JIRA ticket
git checkout -b feature/WS-123-add-ecs-service
git push -u origin feature/WS-123-add-ecs-service

# Branch naming convention: feature/[TICKET-ID]-[brief-description]
# Examples:
# feature/WS-123-add-ecs-service
# feature/WS-124-configure-alb
# feature/WS-125-setup-rds
```

### Commit and Push Frequency
- **Minimum**: Push changes once per day
- **Preferred**: Multiple commits and pushes throughout the day
- **Commit messages**: Clear, concise, reference ticket number
```bash
git commit -m "WS-123: Add ECS task definition with Free Tier limits"
git commit -m "WS-123: Configure ALB target group for ECS service"
git push origin feature/WS-123-add-ecs-service
```

### Pull Request Strategy  
- Create PRs at logical completion points
- Multiple PRs per ticket allowed for easier review
- Each PR should be focused and reviewable
```bash
gh pr create --title "WS-123: Add ECS task definition" --body "Part 1 of ECS service implementation"
gh pr create --title "WS-123: Configure load balancer" --body "Part 2 of ECS service implementation"
```

### Pre-commit Requirements
- Terraform formatting and validation
- Security scanning with tfsec
- Conventional commit messages with ticket reference
- No secrets detection

### Documentation Creation Process
1. **Research**: Create Confluence page in "Software Development" space under `ecs-fargate-infra/research/`
2. **Technical Design**: Document in `docs/architecture/` with ADRs in `docs/decisions/`
3. **Task Management**: Create JIRA tickets in WebStore project for implementation tracking
4. **Code Changes**: Use `gh` CLI with `awynne` account for PR creation and management
5. **Language/Framework Questions**: Query context7 MCP for syntax and best practices

### Cost Management Checklist
- [ ] Verify Free Tier eligibility  
- [ ] Set AWS Budget ($10 limit, 80% alert)
- [ ] Configure billing email alerts
- [ ] Monitor usage weekly with scripts
- [ ] Use minimal resource configurations
- [ ] 7-day CloudWatch log retention
- [ ] Scale down when not developing

### Free Tier Limitations
- **Cross-AZ data transfer**: Use single AZ
- **NAT Gateway**: $45/month - use public subnets  
- **RDS snapshots**: Count against storage limits
- **Custom CloudWatch metrics**: Limited to 10 free
- **Unattached Elastic IPs**: $0.005/hour charge

### Alternative Approaches
1. **Serverless**: Lambda + API Gateway + DynamoDB
2. **EC2**: Single t2.micro with Docker Compose  
3. **Static + Lambda**: S3 + CloudFront + Lambda
4. **Lightsail**: Fixed $3.50/month pricing

## Tool Access
### GitHub CLI
- Account: `awynne`
- Usage: `gh pr create`, `gh pr status`, `gh pr merge`

### JIRA Integration  
- Project: WebStore
- Access: Via configured MCP server
- Usage: Task creation and status tracking

### Confluence Documentation
- Space: "Software Development"  
- Page Tree: `ecs-fargate-infra/`
- Access: Via Atlassian MCP server
- Usage: Research documentation and spike findings

### Language Documentation
- Service: context7 MCP connection
- Usage: Syntax reference and framework documentation for current versions

## References
- [AWS Free Tier Details](https://aws.amazon.com/free/)
- [AWS Cost Calculator](https://calculator.aws/)
- Technical documentation: `docs/` directory
- Jira project is located at https://awynnejira.atlassian.net
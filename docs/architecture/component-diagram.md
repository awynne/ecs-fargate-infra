# Component Diagram - ECS Fargate Infrastructure for Ruby on Rails

## Overview

This document provides detailed component diagrams showing the relationships and interactions between different parts of the ECS Fargate infrastructure optimized for **Ruby on Rails applications** with **PostgreSQL database**.

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
│  │  │  │   (Port 80/443) │────│ Rails App Tasks │        │  │  │
│  │  │  │                 │    │  (Port 3000)    │        │  │  │
│  │  │  └─────────────────┘    └─────────────────┘        │  │  │
│  │  │                                  │                 │  │  │
│  │  └──────────────────────────────────│─────────────────┘  │  │
│  │                                     │                    │  │
│  │  ┌─────────────────────────────────────────────────────┐  │  │
│  │  │              Private Subnet (10.0.2.0/24)          │  │  │
│  │  │                                                     │  │  │
│  │  │  ┌─────────────────────────────────────────────────│  │  │
│  │  │  │             RDS PostgreSQL                      │  │  │
│  │  │  │          (db.t3.micro)                          │  │  │
│  │  │  │            Port 5432                            │  │  │
│  │  │  └─────────────────────────────────────────────────│  │  │
│  │  └─────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────┘  │
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
│   ├── ALB (Application Load Balancer)
│   └── ECS Fargate Tasks (Rails Application)
├── Private Subnet: 10.0.2.0/24 (AZ: us-east-1a)
│   └── RDS PostgreSQL (db.t3.micro)
├── Internet Gateway
└── Route Tables
    ├── Public Route Table (0.0.0.0/0 → IGW)
    └── Private Route Table (local VPC traffic only)
```

#### Security Groups
```
ALB Security Group (alb-sg)
├── Inbound: 0.0.0.0/0:80 (HTTP)
├── Inbound: 0.0.0.0/0:443 (HTTPS)
└── Outbound: ECS-SG:3000 (Rails application)

ECS Security Group (ecs-sg)
├── Inbound: ALB-SG:3000 (Rails application port)
└── Outbound: RDS-SG:5432 (PostgreSQL) + 0.0.0.0/0:443 (AWS APIs)

RDS Security Group (rds-sg)
├── Inbound: ECS-SG:5432 (PostgreSQL)
└── Outbound: None (database doesn't initiate outbound connections)
```

### Compute Components

#### ECS Cluster Architecture
```
ECS Cluster: rails-fargate-cluster
│
├── Service: rails-web-service
│   ├── Task Definition: rails-app:latest
│   │   ├── CPU: 256 units (0.25 vCPU)
│   │   ├── Memory: 512 MB
│   │   ├── Container: rails-app
│   │   │   ├── Image: {account}.dkr.ecr.{region}.amazonaws.com/rails-app:latest
│   │   │   ├── Port: 3000 (Rails default)
│   │   │   ├── Environment Variables:
│   │   │   │   ├── RAILS_ENV=production
│   │   │   │   ├── DATABASE_URL (from Parameter Store)
│   │   │   │   └── SECRET_KEY_BASE (from Parameter Store)
│   │   │   └── Health Check: GET /health
│   │   └── IAM Task Role: ecs-rails-task-role
│   │       ├── RDS PostgreSQL access
│   │       ├── Parameter Store access (database credentials)
│   │       └── CloudWatch Logs access
│   │
│   ├── Desired Count: 1 (development), 0 (off-hours)
│   ├── Auto Scaling Target: 1-3 tasks
│   ├── Health Check: Rails /health endpoint
│   └── Load Balancer Integration: Target Group registration
│
└── Service Discovery: None (cost optimization)
```

#### Application Load Balancer
```
ALB: rails-fargate-alb
│
├── Listener: HTTP:80
│   └── Action: Redirect to HTTPS
│
├── Listener: HTTPS:443
│   ├── SSL Certificate: AWS Certificate Manager
│   └── Target Group: rails-tasks-tg
│       ├── Protocol: HTTP:3000 (Rails application)
│       ├── Health Check: GET /health (Rails health endpoint)
│       ├── Health Check Interval: 30s
│       ├── Healthy Threshold: 2
│       ├── Unhealthy Threshold: 5
│       └── Health Check Timeout: 10s
│
└── Access Logs: Disabled (cost optimization)
```

### Data Components

#### RDS PostgreSQL Configuration
```
RDS Instance: rails-postgres-db
│
├── Engine: PostgreSQL 15.x (latest in Free Tier)
├── Instance Class: db.t3.micro (1 vCPU, 1GB RAM)
├── Storage: 20GB General Purpose SSD (gp2)
├── Multi-AZ: Disabled (single AZ for cost optimization)
├── Backup Configuration:
│   ├── Backup Retention: 7 days (minimum)
│   ├── Backup Window: 03:00-04:00 UTC (off-hours)
│   └── Backup Storage: 20GB (included in Free Tier)
├── Maintenance Window: Sunday 04:00-05:00 UTC
├── Encryption: Disabled (to stay in Free Tier)
├── Performance Insights: Disabled (cost optimization)
└── Enhanced Monitoring: Disabled (cost optimization)
```

**Database Schema**: 
Rails ActiveRecord will manage schema through migrations:
```sql
-- Example Rails-managed tables
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  email VARCHAR NOT NULL UNIQUE,
  created_at TIMESTAMP NOT NULL,
  updated_at TIMESTAMP NOT NULL
);

CREATE TABLE posts (
  id SERIAL PRIMARY KEY,
  title VARCHAR NOT NULL,
  content TEXT,
  user_id INTEGER REFERENCES users(id),
  created_at TIMESTAMP NOT NULL,
  updated_at TIMESTAMP NOT NULL
);
```

### Supporting Services

#### CloudWatch Configuration
```
CloudWatch
│
├── Log Groups
│   ├── /ecs/rails-app (7-day retention)
│   ├── /aws/rds/instance/rails-postgres-db/postgresql (7-day retention)
│   ├── /aws/applicationloadbalancer/access (disabled for cost)
│   └── Rails Application Logs (via Rails logger)
│
├── Metrics
│   ├── ECS Service Metrics (CPU, Memory, Task Count)
│   ├── ALB Metrics (RequestCount, TargetResponseTime, HTTPCode_Target_2XX)
│   ├── RDS Metrics (CPUUtilization, DatabaseConnections, FreeStorageSpace)
│   └── Custom Rails Metrics (limited to 10 for Free Tier)
│
└── Alarms
    ├── ECS CPU > 80% (scale up Rails tasks)
    ├── ECS CPU < 20% (scale down Rails tasks)
    ├── RDS CPU > 80% (performance alert)
    ├── RDS Connections > 15 (connection pool alert)
    ├── ALB 5xx errors > 10/minute (Rails application errors)
    ├── RDS Free Tier hours > 600/month (80% threshold)
    └── Monthly cost > $8 (80% of budget)
```

#### Systems Manager Parameter Store
```
Parameter Store
│
├── /rails/production/database_url (SecureString)
│   └── postgresql://username:password@host:5432/database
├── /rails/production/secret_key_base (SecureString)
├── /rails/production/rails_env (String) → "production"
├── /rails/production/log_level (String) → "info"
└── /rails/production/database_pool_size (String) → "5"
```

#### Elastic Container Registry
```
ECR Repository: rails-app
├── URI: {account}.dkr.ecr.{region}.amazonaws.com/rails-app
├── Lifecycle Policy: Keep last 10 images (cost optimization)
├── Vulnerability Scanning: Enabled
└── Image Tags: latest, v1.0.0, v1.0.1, etc.
```

## Data Flow Diagrams

### Request Processing Flow
```
Internet User
     │ HTTPS (443)
     ▼
Application Load Balancer
     │ HTTP (3000)
     ▼
ECS Fargate Task (Rails App)
     │ ActiveRecord/pg gem
     ▼
RDS PostgreSQL
     │ SQL Result Set
     ▼
ECS Fargate Task (Rails App)
     │ Rails Response (JSON/HTML)
     ▼
Application Load Balancer
     │ HTTPS Response
     ▼
Internet User
```

### Deployment Flow
```
Developer
     │ docker build & push
     ▼
ECR Repository (rails-app)
     │ new Rails image
     ▼
ECS Service Update
     │ rolling deployment
     ▼
New Rails Task Definition
     │ Rails health check (/health)
     ▼
Target Group Registration
     │ ALB traffic routing
     ▼
Active Rails Application

Database Migration Flow:
```
Rails Container Startup
     │ bundle exec rails db:migrate
     ▼
RDS PostgreSQL
     │ schema updates
     ▼
Rails Application Ready
```
```

### Auto Scaling Flow
```
CloudWatch Metrics
     │ Rails App CPU > 80%
     ▼
ECS Auto Scaling Policy
     │ scale out Rails tasks
     ▼
ECS Service
     │ launch new Rails task
     ▼
Task Placement (Public Subnet)
     │ register with ALB Target Group
     ▼
Load Distribution Across Rails Tasks
```

### Cost Optimization Scheduling
```
CloudWatch Events (Cron)
     │ 
     ├── 9 AM Weekdays: Start RDS + Scale ECS to 1
     ├── 6 PM Weekdays: Scale ECS to 0 + Stop RDS  
     └── Weekends: Keep RDS stopped, ECS at 0
```

## Security Architecture

### IAM Roles and Policies
```
ECS Task Execution Role (ecs-rails-execution-role)
├── AmazonECSTaskExecutionRolePolicy
├── ECR access (pull Rails images)
├── CloudWatch Logs access
└── Parameter Store read access (for environment variables)

ECS Task Role (ecs-rails-task-role)
├── RDS Connect access (specific database)
├── Parameter Store read access (database credentials)
├── CloudWatch custom metrics
└── CloudWatch Logs write access

RDS Enhanced Monitoring Role
└── (Managed by AWS, if enabled)

ALB Service Role
└── (Managed by AWS)
```

### Network Security
```
Defense in Depth
│
├── Edge: ALB with SSL termination (WAF optional, cost consideration)
├── Network: Security Groups (stateful firewall rules)
├── Application: Rails application security (CSRF, XSS protection)
├── Database: RDS in private subnet, encrypted connections
└── Data: RDS encryption at rest (optional, additional cost)
```

## Cost Optimization Components

### Free Tier Usage Tracking
```
Cost Monitoring
│
├── ECS Fargate: 20GB-hours/month (Rails tasks)
├── RDS PostgreSQL: 750 hours/month db.t3.micro
├── RDS Storage: 20GB + 20GB backup
├── ALB: 750 hours + 15GB processing
├── CloudWatch: 5GB logs + 10 metrics
└── Parameter Store: 10,000 parameters
```

### Resource Scheduling
```
Development Schedule (Cost Optimized)
│
├── Business Hours (9 AM - 6 PM Weekdays):
│   ├── RDS: Running
│   └── ECS: Min 1, Max 3 Rails tasks
│
├── Off Hours (6 PM - 9 AM Weekdays):
│   ├── RDS: Stopped (save Free Tier hours)
│   └── ECS: Min 0, Max 0 tasks
│
└── Weekends:
    ├── RDS: Stopped
    └── ECS: Min 0, Max 0 tasks
```

This component architecture provides a comprehensive view of how all pieces fit together while maintaining cost optimization and Free Tier compliance.
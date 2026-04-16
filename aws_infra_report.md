# 🏗️ AWS Infrastructure Reference — aifa Project
> **Account:** `445567111055` | **Region:** `ap-southeast-1` (Singapore)
> **Fetched:** 2026-04-15 | **Source:** Live AWS API

---

## 📋 Overview Matrix

| Service | DEV ✅ (live — chuẩn) | STAGING 🎯 (desired — prod-ready) |
|---|---|---|
| **VPC CIDR** | `172.32.0.0/16` | `172.33.0.0/16` |
| **NAT Gateway** | 1 × AZ-a | 1 × AZ-a |
| **ECS BE** | 1 vCPU / 2 GB | **2 vCPU / 8 GB** |
| **ECS MCP** | 0.5 vCPU / 1 GB | **1 vCPU / 2 GB** |
| **ECS Admin** | 0.25 vCPU / 1 GB (stopped) | **1 vCPU / 2 GB** |
| **ECS Celery** | ❌ | ❌ **Bỏ** |
| **Redis** | EC2 `t4g.micro` | **EC2 `t4g.medium`** |
| **RDS Engine** | PostgreSQL 17.4 | PostgreSQL 17.4 |
| **RDS Instance** | `db.t4g.micro` | **`db.t4g.large`** |
| **RDS Storage** | 20 GB gp2 | **100 GB gp3** |
| **RDS Backup** | 1 day | **7 days** |
| **RDS Deletion Prot.** | ❌ | **✅ ON** |
| **RDS Perf Insights** | ❌ | **✅ ON (7d)** |
| **S3 Bucket** | `aifa-dev-public-bucket` | `aifa-staging-public-bucket` |
| **SQS** | ✅ `aifa-run-process-queue-dev` | ✅ **`aifa-run-process-queue-stg`** |
| **Lambda task runner** | ✅ 3008 MB / ARM64 / SQS | ✅ **3008 MB / ARM64 / SQS** |
| **Lambda file pipeline** | ✅ 3008 MB / ARM64 | ✅ **3008 MB / ARM64** |
| **WS Throttle Burst** | 5,000 | **5,000** |
| **WS Throttle Rate** | 10,000 rps | **10,000 rps** |
| **Bastion** | `t4g.nano` | `t4g.nano` |

---

## 1. 🔐 Amazon Cognito

| Config | DEV ✅ | STAGING 🎯 |
|---|---|---|
| **Pool Name** | `aifa-dev-user-congito-pool` | `aifa-staging-user-congito-pool` |
| **Pool ID** | `ap-southeast-1_pbErwjty3` | `ap-southeast-1_uZSuLUEHV` |
| **Client Name** | `aifa-dev-app-client` | `aifa-staging-app-client` |
| **Client ID** | `45qrn1ejsr0m9mantr43po8uov` | `4mkhotsa9fm07jk4n4221mg0mo` |
| **MFA** | OFF | OFF |
| **Access Token TTL** | 1 day | 1 day |
| **ID Token TTL** | 1 day | 1 day |
| **Refresh Token TTL** | 5 days | 5 days |
| **Auth Flows** | `ALLOW_REFRESH_TOKEN_AUTH` `ALLOW_USER_PASSWORD_AUTH` `ALLOW_USER_SRP_AUTH` | `ALLOW_REFRESH_TOKEN_AUTH` `ALLOW_USER_PASSWORD_AUTH` `ALLOW_USER_SRP_AUTH` |
| **PostConfirmation Lambda** | `aifa-dev-create-account` | `aifa-stg-create-account` |
| **Email Sending** | COGNITO_DEFAULT | DEVELOPER |
| **Email Source** | `no-reply@aifa.vn` (SES) | `no-reply@aifa.vn` (SES) |
| **Password Policy** | Min 8, Upper+Lower+Num+Symbol | Min 8, Upper+Lower+Num+Symbol |
| **Callback URL** | `https://d84l1y8p4kdic.cloudfront.net` | `https://d84l1y8p4kdic.cloudfront.net` |

---

## 2. 🌐 API Gateway

### REST API
| Config | DEV ✅ | STAGING 🎯 |
|---|---|---|
| **Name** | `aifa-dev-rest-api` | `aifa-staging-rest-api` |
| **ID** | `z7nfi76aw9` | `15tkcxwcpg` |
| **Type** | REGIONAL | REGIONAL |
| **Stage** | `v1` | `v1` |
| **Resources count** | 11 | 11 |
| **Cache** | Disabled | Disabled |
| **Throttle** | Default | Default |

### WebSocket API
| Config | DEV ✅ | STAGING 🎯 |
|---|---|---|
| **Name** | `aifa-dev-websocket` | `aifa-staging-websocket` |
| **ID** | `7y9xf9299c` | `88nwpbtob6` |
| **Endpoint** | `wss://7y9xf9299c.execute-api.ap-southeast-1.amazonaws.com` | `wss://88nwpbtob6.execute-api.ap-southeast-1.amazonaws.com` |
| **Stage** | `development` | `staging` |
| **Route Selection Expr** | `$request.body.action` | `$request.body.action` |
| **Throttle Burst** | 5,000 | **5,000** |
| **Throttle Rate** | 10,000 rps | **10,000 rps** |
| **Data Trace** | Enabled | Enabled |
| **Logging Level** | INFO | INFO |

---

## 3. 🔗 VPC & Networking

### DEV VPC — `vpc-06a1c6d8ae38672b6` · `172.32.0.0/16`
| Subnet | ID | CIDR | AZ | Type |
|---|---|---|---|---|
| public1 | `subnet-0fa7165b3c7cc1ee0` | `172.32.0.0/20` | ap-se-1a | Public |
| public2 | `subnet-05074a728f0fa0a5d` | `172.32.16.0/20` | ap-se-1b | Public |
| public3 | `subnet-061ade89954d4b326` | `172.32.32.0/20` | ap-se-1c | Public |
| private1 | `subnet-09a2f2591f7b646e9` | `172.32.128.0/20` | ap-se-1a | Private |
| private2 | `subnet-0974ea5b1f4cd0826` | `172.32.144.0/20` | ap-se-1b | Private |
| private3 | `subnet-03d12ad1ee24e07db` | `172.32.160.0/20` | ap-se-1c | Private |

**NAT:** `nat-06ebf74670c07fa79` · `54.151.248.186` · AZ-a
**S3 Endpoint:** ×2 Gateway (no NAT cost)
**Security Groups (11):** `dev-aifa-bastion`, `aifa-dev-rds-sg`, `aifa-dev-nlb-sg`, `aifa-dev-ecs-sg`, `aifa-dev-lambda-sg`, `aifa-admin-dev-alb-sg`, `aifa-dev-endpoint-sg`, `aifa-dev-ec2-sg`, `aifa-dev-alb-sg`, `aifa-dev-vpc-link-sg`, `aifa-dev-redis-sg`

### STAGING VPC — `vpc-0f1d2898e5f65ffd6` · `172.33.0.0/16`
| Subnet | ID | CIDR | AZ | Type |
|---|---|---|---|---|
| public1 | `subnet-0a1401041acc8aa54` | `172.33.0.0/20` | ap-se-1a | Public |
| public2 | `subnet-00db5e52a2027c1a1` | `172.33.16.0/20` | ap-se-1b | Public |
| public3 | `subnet-04126dbac7892e384` | `172.33.32.0/20` | ap-se-1c | Public |
| private1 | `subnet-00814588b33791444` | `172.33.128.0/20` | ap-se-1a | Private |
| private2 | `subnet-0ca065fac02e537d6` | `172.33.144.0/20` | ap-se-1b | Private |
| private3 | `subnet-06d1ea68a2379cbc5` | `172.33.160.0/20` | ap-se-1c | Private |

**NAT:** `nat-01face0c2c3f36a62` · `18.139.99.214` · AZ-a
**Security Groups (8):** `aifa-staging-vpc-link-sg`, `aifa-staging-endpoint-sg`, `aifa-stg-lambda-sg`, `aifa-staging-nlb-sg`, `aifa-staging-alb-sg`, `aifa-staging-redis-sg`, `aifa-staging-ecs-sg`, `aifa-staging-rds-sg`

---

## 4. ⚖️ Load Balancers

### NLB
| Config | DEV ✅ | STAGING 🎯 |
|---|---|---|
| **Name** | `aifa-dev-nlb` | `aifa-staging-nlb-2` |
| **Scheme** | internet-facing | internet-facing |
| **DNS** | `aifa-dev-nlb-72cbc336c95ec0e6.elb.ap-southeast-1.amazonaws.com` | `aifa-staging-nlb-2-a481933708f1b67e.elb.ap-southeast-1.amazonaws.com` |
| **AZs** | 1a, 1b, 1c | 1a, 1b, 1c |
| **Listener** | TCP :80 | TCP :80 |
| **Target Group** | `aifa-dev-nlb-tg` · TCP :80 · ip | `aifa-staging-nlb-tg` · TCP :80 · ip |
| **Health Check** | HTTP `/health` · 30s · threshold=5 | HTTP `/health` · 30s · threshold=5 |

### ALB (Admin)
| Config | DEV ✅ | STAGING 🎯 |
|---|---|---|
| **Name** | `aifa-admin-dev-alb` | `aifa-admin-staging-alb` |
| **Scheme** | internet-facing | internet-facing |
| **DNS** | `aifa-admin-dev-alb-1434171630.ap-southeast-1.elb.amazonaws.com` | `aifa-admin-staging-alb-1127654142.ap-southeast-1.elb.amazonaws.com` |
| **AZs** | 1b, 1c | 1a, 1b, 1c |
| **Listeners** | HTTPS :443 + HTTP :80 | HTTPS :443 + HTTP :80 |
| **TLS Policy** | `ELBSecurityPolicy-TLS13-1-2-Res-2021-06` | `ELBSecurityPolicy-TLS13-1-2-Res-2021-06` |
| **Target Group** | `aifa-admin-http-tg-dev` · HTTP :80 | `aifa-admin-http-tg-staging` · HTTP :80 |
| **Health Check** | HTTP `/health` · 30s · threshold=5 | HTTP `/health` · 30s · threshold=5 |

---

## 5. 🐳 ECS Fargate

### Cluster BE

| Config | DEV ✅ `aifa-be-dev-cluster` | STAGING 🎯 `aifa-be-staging-cluster` |
|---|---|---|
| **BE Service** | `aifa-dev-be-public-service` | `aifa-staging-be-public-service` |
| **BE Image** | `aifa-dev-backend:v20260413-021647` | `aifa-staging-backend:latest` |
| **BE CPU / Memory** | 1024 / 2048 MB | **2048 / 8192 MB** |
| **BE Port** | 5000 | 5000 |
| **BE Task Def** | `aifa-dev-task-definition` rev.449 | `aifa-staging-task-definition` |
| **BE Env Vars** | 62 | 62+ |
| **BE LB** | NLB → :5000 | NLB → :5000 |
| **MCP Service** | `aifa-dev-mcp-service` | `aifa-staging-mcp-service` |
| **MCP Image** | `aifa-dev-mcp:v20260413-021643` | `aifa-staging-mcp:latest` |
| **MCP CPU / Memory** | 512 / 1024 MB | **1024 / 2048 MB** |
| **MCP Port** | 8765 | 8765 |
| **MCP Task Def** | `aifa-dev-mcp-task-definition` rev.11 | `aifa-staging-mcp-task-definition` |
| **MCP Env Vars** | 63 | 63+ |
| **Celery** | ❌ | ❌ **Bỏ** |
| **Circuit Breaker** | enable + rollback | enable + rollback |
| **Deploy Rolling** | max=200% · min=100% | max=200% · min=100% |
| **Logging** | awslogs (CloudWatch) | awslogs (CloudWatch) |
| **Network Mode** | awsvpc | awsvpc |

### Cluster Admin

| Config | DEV ✅ `aifa-admin-be-dev` | STAGING 🎯 `aifa-admin-be-staging` |
|---|---|---|
| **Service** | `aifa-admin-dev-service` | `aifa-admin-staging-service` |
| **Image** | `aifa-admin-web-be:v20251203-085019` | `aifa-admin-web-be:latest` |
| **CPU / Memory** | 256 / 1024 MB | **1024 / 2048 MB** |
| **Port** | 8000 | 8000 |
| **Task Def** | `aifa-admin-dev` rev.21 | `aifa-admin-staging` |
| **LB** | ALB → :8000 | ALB → :8000 |
| **Circuit Breaker** | enable + rollback | enable + rollback |

---

## 6. ⚡ Lambda Functions

| Function | DEV ✅ | STAGING 🎯 |
|---|---|---|
| **Task Runner** | `dev-process-task-runner` | `stg-process-task-runner` |
| ↳ Runtime | Image | Image |
| ↳ Memory | **3008 MB** | **3008 MB** |
| ↳ Timeout | **900s** | **900s** |
| ↳ Arch | **ARM64** | **ARM64** |
| ↳ Trigger | SQS batch=10 · window=5s | SQS batch=10 · window=5s |
| ↳ VPC | private1/2/3 · lambda-sg | private1/2/3 · stg-lambda-sg |
| ↳ Env Vars | 33 keys | 33 keys (stg values) |
| **File Pipeline** | `file-upload-pipeline-vector-isolate-dev` | `file-upload-pipeline-vector-isolate-stg` |
| ↳ Runtime | Image | Image |
| ↳ Memory | **3008 MB** | **3008 MB** |
| ↳ Timeout | **900s** | **900s** |
| ↳ Arch | **ARM64** | **ARM64** |
| ↳ Trigger | Sync invoke | Sync invoke |
| ↳ Env Vars | 23 keys | 23 keys (stg values) |
| **Create Account** | `aifa-dev-create-account` | `aifa-stg-create-account` |
| ↳ Runtime | python3.12 | python3.12 |
| ↳ Memory | 128 MB | 128 MB |
| ↳ Timeout | 300s | 300s |
| ↳ Trigger | Cognito PostConfirmation | Cognito PostConfirmation |
| **Downgrade User** | `downgrade_user` | `downgrade_user_stg` |
| ↳ Runtime | python3.12 | python3.12 |
| ↳ Memory | 128 MB | 128 MB |
| ↳ Timeout | 3s | 3s |
| ↳ Trigger | EventBridge every-10-mins | EventBridge every-10-mins |
| **Process Cleanup** | `cleanup-process-tasks` | `aifa-report-process-cleanup` |
| ↳ Runtime | python3.12 | python3.11 |
| ↳ Memory | 128 MB | 256 MB |
| ↳ Timeout | 300s | 300s |
| ↳ Trigger | EventBridge cron 2AM | EventBridge |
| **Autoscaling** | — | `autoscaling_function_stg` |

### `dev-process-task-runner` — Env Vars (33 keys)
```
REDIS_DB, REDIS_HOST, REDIS_PORT, REDIS_PASSWORD
MCP_SSE_URL
DB_PORT, DB_USER, DB_NAME, DB_HOST, DB_PASSWORD
ENVIRONMENT
LANGFUSE_BASE_URL, LANGFUSE_SECRET_KEY, LANGFUSE_PUBLIC_KEY
VERTEX_MODEL, VERTEX_REGION, VERTEX_TEMPERATURE
VERTEX_NON_THINK_MODEL, VERTEX_NON_THINK_REGION, VERTEX_NON_THINK_TEMPERATURE
LLM_DEEPSEEK_API_KEY, LLM_DEEPSEEK_NAME
SENTRY_DSN, SENTRY_ENVIRONMENT, SENTRY_TRACES_SAMPLE_RATE
WEBSOCKET_ENDPOINT
S3_BUCKET_NAME, AWS_S3_CHROMADB_PREFIX
GCP_REGION_NAME, VERTEXAI_3RD_SA_INFO_DICT
CHROMADB_LOCAL_PATH
HOME
LLM_OPENAI_KEY
```

### `file-upload-pipeline-vector-isolate-dev` — Env Vars (23 keys)
```
DB_PORT, DB_USER, DB_NAME, DB_HOST, DB_PASSWORD
SQLALCHEMY_DATABASE_URI
GEMINI_API_KEY, GEMINI_NAME (LLM_MODEL_NAME)
VERTEXAI_LOCATION, VERTEXAI_3RD_SA_INFO_DICT
LANGFUSE_BASE_URL, LANGFUSE_PUBLIC_KEY, LANGFUSE_SECRET_KEY
SENTRY_DSN, SENTRY_ENVIRONMENT, SENTRY_TRACES_SAMPLE_RATE
USE_NEW_PIPELINE, TASK_QUEUE_TYPE
S3_BUCKET_NAME
ENV
LLM_OPENAI_KEY
WEBSOCKET_ENDPOINT
```

---

## 7. 🗄️ RDS PostgreSQL

| Config | DEV ✅ | STAGING 🎯 |
|---|---|---|
| **Identifier** | `aifa-dev-rds-instance` | `aifa-staging-rds-instance-3` |
| **Engine** | PostgreSQL 17.4 | PostgreSQL 17.4 |
| **Instance Class** | `db.t4g.micro` (2 vCPU / 1 GB) | **`db.t4g.large`** (2 vCPU / 8 GB) |
| **Storage** | 20 GB / gp2 | **100 GB / gp3** |
| **Storage Autoscale** | ❌ | **✅ → max 200 GB** |
| **IOPS** | 3000 (gp2 baseline) | **3000 (gp3 free baseline)** |
| **AZ** | ap-se-1b | ap-se-1b |
| **Multi-AZ** | ❌ | ❌ |
| **Endpoint** | `aifa-dev-rds-instance.c90iewywoamn.ap-southeast-1.rds.amazonaws.com:5432` | `aifa-staging-rds-instance-3.c90iewywoamn.ap-southeast-1.rds.amazonaws.com:5432` |
| **Master User** | `postgres` | `postgres` |
| **Backup Retention** | 1 day | **7 days** |
| **Deletion Protection** | ❌ | **✅ ON** |
| **Publicly Accessible** | ❌ | ❌ |
| **Performance Insights** | ❌ | **✅ ON (7-day free)** |
| **Auto Minor Upgrade** | ✅ ON | **❌ OFF** |
| **Parameter Group** | `default.postgres17` | `default.postgres17` |
| **Subnet Group** | `aifa-dev-rds-subnet-group` | `aifa-staging-rds-subnet-group` |
| **Security Group** | `sg-061d4f2f7c51f486e` (aifa-dev-rds-sg) | `sg-0c276b08550e30210` (aifa-staging-rds-sg) |

---

## 8. 🪣 S3 Buckets

| Config | DEV ✅ | STAGING 🎯 |
|---|---|---|
| **Bucket** | `aifa-dev-public-bucket` | `aifa-staging-public-bucket` |
| **Region** | ap-southeast-1 | ap-southeast-1 |
| **Versioning** | Disabled | Disabled |
| **Encryption** | AES256 | AES256 |
| **CORS Rules** | 1 | 1 |
| **Lifecycle** | None | None |
| **Public Access Block** | ✅ All blocked | ✅ All blocked |
| **S3 VPC Endpoint** | ×2 Gateway | ×2 Gateway |

---

## 9. 📬 SQS

| Config | DEV ✅ | STAGING 🎯 |
|---|---|---|
| **Main Queue** | `aifa-run-process-queue-dev` | **`aifa-run-process-queue-stg`** |
| ↳ Type | Standard | Standard |
| ↳ Visibility Timeout | **900s** | **900s** |
| ↳ Message Retention | 14 days | 14 days |
| ↳ Max Message Size | 1 MB | 1 MB |
| ↳ Receive Wait Time | 0s | 0s |
| ↳ DLQ | `aifa-run-process-queue-dev-dlq` | **`aifa-run-process-queue-stg-dlq`** |
| ↳ DLQ Max Receives | 3 | 3 |
| **DLQ** | `aifa-run-process-queue-dev-dlq` | **`aifa-run-process-queue-stg-dlq`** |
| ↳ Visibility Timeout | 30s | 30s |
| ↳ Message Retention | 14 days | 14 days |

---

## 10. ⏰ EventBridge Rules

| Rule | Schedule | Target | Env |
|---|---|---|---|
| `cleanup-process-tasks-schedule-dev` | `cron(0 2 * * ? *)` | `cleanup-process-tasks` Lambda | DEV |
| `cleanup-process-tasks-schedule-stg` | `cron(0 2 * * ? *)` | `cleanup-process-tasks-stg` Lambda | **STAG 🎯** |
| `every-10-mins` | `rate(10 minutes)` | `downgrade_user`, `downgrade_user_stg`, `autoscaling_function_stg`, `aifa-report-process-cleanup` | Shared |
| `aifa-report-process-cleanup-schedule` | `rate(10 minutes)` | `aifa-report-process-cleanup` | STAG |

---

## 11. 🖥️ EC2 (Redis + Bastion)

### Redis
| Config | DEV ✅ | STAGING 🎯 |
|---|---|---|
| **Instance Name** | `redis-dev` | `redis-stag` |
| **Instance ID** | `i-0984bb33e94d7ea2f` | `i-000feaeb409764365` |
| **Instance Type** | `t4g.micro` (2 vCPU / 1 GB) | **`t4g.medium`** (2 vCPU / 4 GB) |
| **State** | running | running |
| **Private IP** | `172.32.131.22` | `172.33.143.96` |
| **Public IP** | None | None |
| **AZ** | ap-se-1a | ap-se-1a |
| **VPC** | DEV VPC | STAG VPC |
| **Security Group** | `redis-dev-ec2-sg` | `redis-stg-ec2-sg` |

### Bastion
| Config | DEV ✅ | STAGING 🎯 |
|---|---|---|
| **Instance Name** | `dev-aifa-bastion` | `aifa-staging-bastion` |
| **Instance ID** | `i-0c2dcd6b0d89e3b45` | `i-0caecb0b1b72f05ee` |
| **Instance Type** | `t4g.nano` | `t4g.nano` |
| **Private IP** | `172.32.21.249` | `172.33.2.83` |
| **Public IP** | `18.139.115.115` | `54.254.139.162` |
| **AZ** | ap-se-1b | ap-se-1a |

---

## 12. 🔒 Security Groups

### DEV VPC — 11 SGs
| SG Name | ID | In | Out |
|---|---|---|---|
| `dev-aifa-bastion` | `sg-0134154c601f6a40f` | 1 | 1 |
| `aifa-dev-rds-sg` | `sg-061d4f2f7c51f486e` | 1 | 1 |
| `aifa-dev-nlb-sg` | `sg-0c01e1e233fa32260` | 1 | 1 |
| `aifa-dev-ecs-sg` | `sg-006728f9b136af2af` | 4 | 1 |
| `aifa-dev-lambda-sg` | `sg-0278ac383338be469` | 0 | 1 |
| `aifa-admin-dev-alb-sg` | `sg-0329bbf11fe8629c6` | 2 | 1 |
| `aifa-dev-endpoint-sg` | `sg-0b1f0e89bf017f688` | 1 | 1 |
| `aifa-dev-ec2-sg` | `sg-055a9190ea5c10f5c` | 1 | 1 |
| `aifa-dev-alb-sg` | `sg-0289f3186c4fd076a` | 1 | 1 |
| `aifa-dev-vpc-link-sg` | `sg-013581fa6d257e65b` | 1 | 1 |
| `aifa-dev-redis-sg` | `sg-0f9ec5291c7f04828` | 1 | 1 |

### STAGING VPC — 8 SGs
| SG Name | ID | In | Out |
|---|---|---|---|
| `aifa-staging-vpc-link-sg` | `sg-03b15be81574173c3` | 1 | 1 |
| `aifa-staging-endpoint-sg` | `sg-090911eb8bbf444da` | 1 | 1 |
| `aifa-stg-lambda-sg` | `sg-0d2cb77930638028e` | 0 | 1 |
| `aifa-staging-nlb-sg` | `sg-044c5791bdf318b35` | 1 | 1 |
| `aifa-staging-alb-sg` | `sg-0b83f96858ac704c0` | 2 | 1 |
| `aifa-staging-redis-sg` | `sg-052f1c90eb6bafb3d` | 1 | 1 |
| `aifa-staging-ecs-sg` | `sg-05339fa4ed76dd814` | 3 | 1 |
| `aifa-staging-rds-sg` | `sg-0c276b08550e30210` | 1 | 1 |

---

## 📌 STAGING 🎯 — Full Spec Sheet (Quick Reference)

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
NETWORK
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
VPC:        vpc-0f1d2898e5f65ffd6  |  172.33.0.0/16
NAT:        nat-01face0c2c3f36a62  |  18.139.99.214  |  AZ-a
Bastion:    t4g.nano  |  54.254.139.162  (AZ-a)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
COMPUTE — ECS FARGATE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Cluster: aifa-be-staging-cluster
  ├── aifa-staging-be-public-service
  │     CPU: 2048  |  Memory: 8192 MB  |  Port: 5000  |  LB: NLB
  ├── aifa-staging-mcp-service
  │     CPU: 1024  |  Memory: 2048 MB  |  Port: 8765  |  No LB
  └── [❌ Celery — removed]

Cluster: aifa-admin-be-staging
  └── aifa-admin-staging-service
        CPU: 1024  |  Memory: 2048 MB  |  Port: 8000  |  LB: ALB

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
DATABASE — RDS PostgreSQL
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Instance:   db.t4g.large  (2 vCPU / 8 GB RAM)
Storage:    100 GB gp3  |  autoscale → 200 GB
Engine:     PostgreSQL 17.4
Endpoint:   aifa-staging-rds-instance-3.c90iewywoamn.ap-southeast-1.rds.amazonaws.com:5432
Backup:     7 days
Options:    Deletion Protection ON  |  Perf Insights ON (7d)  |  Auto Minor Upgrade OFF

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
CACHE — Redis EC2
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Instance:   t4g.medium  (2 vCPU / 4 GB RAM)
Private IP: 172.33.143.96  (AZ-a, private subnet)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
AUTH — Cognito
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Pool ID:    ap-southeast-1_uZSuLUEHV
Client ID:  4mkhotsa9fm07jk4n4221mg0mo
Token TTL:  Access=1d  |  Refresh=5d
Auth Flows: ALLOW_REFRESH_TOKEN_AUTH + ALLOW_USER_SRP_AUTH + ALLOW_USER_PASSWORD_AUTH

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
API GATEWAY
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
REST:       15tkcxwcpg  |  stage=v1  |  REGIONAL
WebSocket:  88nwpbtob6  |  stage=staging
            wss://88nwpbtob6.execute-api.ap-southeast-1.amazonaws.com/staging
            Throttle: burst=5000  |  rate=10000 rps

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
STORAGE — S3
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Bucket:     aifa-staging-public-bucket
Encryption: AES256  |  CORS: 1 rule  |  Public: blocked

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
QUEUE — SQS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Main:  aifa-run-process-queue-stg
       Visibility=900s  |  Retention=14d  |  DLQ maxReceives=3
DLQ:   aifa-run-process-queue-stg-dlq
       Visibility=30s   |  Retention=14d

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
LAMBDA
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
stg-process-task-runner                  3008MB / 900s / ARM64 / Image / SQS batch=10 window=5s
file-upload-pipeline-vector-isolate-stg  3008MB / 900s / ARM64 / Image / sync
aifa-stg-create-account                  128MB  / 300s / python3.12 / Cognito PostConfirmation
downgrade_user_stg                       128MB  / 3s   / python3.12 / EventBridge
aifa-report-process-cleanup              256MB  / 300s / python3.11 / EventBridge
autoscaling_function_stg                 EventBridge

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
EVENTBRIDGE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
cleanup-process-tasks-schedule-stg   cron(0 2 * * ? *)   cleanup lambda
every-10-mins                        rate(10min)          downgrade_user_stg, autoscaling_function_stg
aifa-report-process-cleanup-schedule rate(10min)          aifa-report-process-cleanup
```

---

## 🔑 Key IDs Reference

| Resource | DEV | STAGING |
|---|---|---|
| VPC | `vpc-06a1c6d8ae38672b6` | `vpc-0f1d2898e5f65ffd6` |
| VPC CIDR | `172.32.0.0/16` | `172.33.0.0/16` |
| Cognito Pool | `ap-southeast-1_pbErwjty3` | `ap-southeast-1_uZSuLUEHV` |
| Cognito Client | `45qrn1ejsr0m9mantr43po8uov` | `4mkhotsa9fm07jk4n4221mg0mo` |
| API GW REST | `z7nfi76aw9` | `15tkcxwcpg` |
| API GW WebSocket | `7y9xf9299c` | `88nwpbtob6` |
| WebSocket URL | `wss://7y9xf9299c.execute-api.ap-southeast-1.amazonaws.com/development` | `wss://88nwpbtob6.execute-api.ap-southeast-1.amazonaws.com/staging` |
| NAT GW | `nat-06ebf74670c07fa79` | `nat-01face0c2c3f36a62` |
| NAT Public IP | `54.151.248.186` | `18.139.99.214` |
| ECS Cluster (BE) | `aifa-be-dev-cluster` | `aifa-be-staging-cluster` |
| ECS Cluster (Admin) | `aifa-admin-be-dev` | `aifa-admin-be-staging` |
| RDS Endpoint | `aifa-dev-rds-instance.c90iewywoamn.ap-southeast-1.rds.amazonaws.com` | `aifa-staging-rds-instance-3.c90iewywoamn.ap-southeast-1.rds.amazonaws.com` |
| RDS Port | `5432` | `5432` |
| Redis Private IP | `172.32.131.22` | `172.33.143.96` |
| Bastion Public IP | `18.139.115.115` | `54.254.139.162` |
| S3 Bucket | `aifa-dev-public-bucket` | `aifa-staging-public-bucket` |
| SQS Main Queue | `aifa-run-process-queue-dev` | `aifa-run-process-queue-stg` |
| Lambda SG | `sg-0278ac383338be469` | `sg-0d2cb77930638028e` |
| ECS SG | `sg-006728f9b136af2af` | `sg-05339fa4ed76dd814` |
| RDS SG | `sg-061d4f2f7c51f486e` | `sg-0c276b08550e30210` |

# NewsPulse


# 📰  **NewsPulse: Real-Time News Aggregator & Recommendation Platform**

### 🎯 Goal

Build a platform that collects live news from multiple sources, processes articles in real time, tags them with NLP (using AWS Comprehend), stores metadata, and serves a personalized news feed via APIs & a dashboard.

---

## ⚙️ Core Architecture

* **Ingestion**

  * Lambda scheduled jobs to fetch RSS/JSON feeds → publish to **Kinesis Stream**
  * Optional WebSocket API for live article submissions
* **Processing**

  * EKS microservice (Python/Node) consuming from Kinesis
  * Uses **AWS Comprehend** for sentiment + entity extraction
  * Writes article metadata to RDS (Postgres)
  * Raw articles stored in S3 (`news-raw`)
* **Serving**

  * REST API (FastAPI/Express) on EKS, behind ALB (TLS via ACM)
  * Frontend (React) hosted on S3 + CloudFront (`app.newspulse.dev`)
* **Recommendations**

  * Simple “most popular in last hour” from Athena queries on S3
  * Stretch: Redis for caching trending results
* **CI/CD**

  * Jenkins CI → build/test/scan images, push to ECR, update Helm values
  * ArgoCD → GitOps sync to EKS
* **Secrets & Security**

  * AWS Secrets Manager + External Secrets Operator
  * IAM roles via IRSA for Kinesis + S3 access
  * WAF rules for API (SQLi/XSS protection)
* **Observability**

  * Prometheus + Grafana + Alertmanager
  * OpenTelemetry traces (API latency, processing times)
  * CloudWatch alarms: ingestion lag, API 5xx
* **Reliability**

  * HPA on API + processor pods
  * Argo Rollouts for canary deploys
  * Chaos testing with LitmusChaos
* **Backups & DR**

  * RDS daily snapshots (14 days retention)
  * S3 lifecycle to IA + Glacier
  * Terraform state in versioned S3

---

## 📂 Repo Layout

```
newspulse/
├── infra/               # Terraform modules: vpc, eks, rds, s3, kinesis, comprehend, sns
├── ops/                 # GitOps manifests (ArgoCD apps, Helm charts, ESO, OPA policies)
└── app/
    ├── api/             # REST API
    ├── processor/       # Kinesis consumer, calls Comprehend
    ├── ingestor/        # Lambda/cron job to fetch RSS
    └── frontend/        # React dashboard
```

---

## 🗓️ One-Week Plan

**Day 1:** Terraform VPC/EKS/RDS/S3/Kinesis/ECR/ACM/Route53
**Day 2:** Install ArgoCD, ESO, Prometheus/Grafana, ALB Ingress Controller
**Day 3:** Set up ingestion (Lambda + Kinesis), RDS schema for articles/users
**Day 4:** Build processor service (consume Kinesis → Comprehend → RDS/S3)
**Day 5:** Build API (query RDS, expose trending news); Jenkins CI pipeline + ECR push
**Day 6:** Deploy frontend (React on S3 + CloudFront); secure API with WAF; Argo Rollouts test
**Day 7:** Observability dashboards, alarms, chaos tests, RDS restore drill, runbooks

---

## 📊 Demo Script

1. Trigger Lambda ingestion → new article enters pipeline
2. Processor enriches article (tags, sentiment) → stored in DB
3. API + frontend shows live feed (`app.newspulse.dev`)
4. Run load test → API scales with HPA, Grafana dashboard updates
5. Deploy faulty build with Argo Rollouts → canary catches error, rollback
6. Trigger RDS restore drill → rotate secret → service recovers

---


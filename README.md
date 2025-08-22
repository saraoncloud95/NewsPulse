# NewsPulse
<img width="1691" height="1610" alt="newspulse" src="https://github.com/user-attachments/assets/32f0f4d4-6f3b-44a0-beef-3625410c1c63" />


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

## ✅ Step 1: Provision Core Infrastructure with Terraform

1. **VPC & Networking**

   <img width="999" height="161" alt="Screenshot from 2025-08-22 11-08-58" src="https://github.com/user-attachments/assets/ed13ad2e-e71f-4563-a00c-0be2ef4cb9b4" />


   * Create a VPC with public/private subnets across 2 AZs.
   * Add Internet Gateway + NAT Gateway.
   * Security groups for ALB, EKS nodes, and RDS.

3. **EKS Cluster**
<img width="1554" height="258" alt="Screenshot from 2025-08-22 11-09-49" src="https://github.com/user-attachments/assets/3fa1977c-e1e6-4e95-82ea-8fb4830d3948" />

   * Spin up an EKS cluster with at least 2 node groups (one for system workloads, one for apps).
   * Enable IAM OIDC provider (needed for IRSA later).
   * Output kubeconfig so you can connect via `kubectl`.

4. **RDS (Postgres)**
<img width="1538" height="236" alt="Screenshot from 2025-08-22 11-25-44" src="https://github.com/user-attachments/assets/0797175c-e270-43f7-b002-02f4ebf2ccf6" />

   * Create RDS Postgres instance (db.t3.medium is fine for demo).
   * Place in private subnets.
   * Store creds in AWS Secrets Manager.

5. **S3 Buckets**
<img width="1048" height="258" alt="Screenshot from 2025-08-22 11-10-33" src="https://github.com/user-attachments/assets/8817548d-3efb-4125-b834-1a557fbf191d" />

   * `news-raw` (store original articles).
   * `news-analytics` (store Athena query data).
   * Enable versioning + lifecycle policies.

6. **Kinesis Stream**
<img width="1564" height="263" alt="Screenshot from 2025-08-22 11-11-05" src="https://github.com/user-attachments/assets/d32c441a-2991-4895-b896-9ef9f36f583a" />

   * Create a stream (`news-ingestion`) with 1–2 shards for now.
   * This will be the pipeline between ingestion Lambda and processor service.

7. **ECR Repository**
<img width="1497" height="344" alt="Screenshot from 2025-08-22 11-11-47" src="https://github.com/user-attachments/assets/dc722d64-a9a1-43d7-a254-f8441eba29d8" />

   * Create repos for `api`, `processor`, and `frontend`.
   * Used by Jenkins CI.

8. **Route53 + ACM**
<img width="1531" height="345" alt="Screenshot from 2025-08-22 11-12-17" src="https://github.com/user-attachments/assets/5da93c4e-c1d3-4527-ad23-5bb33f6add37" />

<img width="1531" height="345" alt="Screenshot from 2025-08-22 11-12-47" src="https://github.com/user-attachments/assets/cd5257f4-90c1-4e82-b365-fd644412081c" />


   * Set up domain `newspulse.dev` (or subdomain if you have one).
   * Request ACM TLS cert for `*.newspulse.dev`.


---

# Step 1: Building the Core Infrastructure for NewsPulse

## Introduction

Every successful digital platform begins with a strong foundation. For **NewsPulse**, a real-time news aggregator and recommendation system, this foundation is built on a set of core AWS services that ensure security, scalability, and reliability. Step 1 focuses on establishing the **networking, compute, storage, and streaming backbone** of the system before moving to application development.

---

## 1. Virtual Private Cloud (VPC) & Networking

A dedicated **VPC** provides a secure, isolated environment for all application components.

* **Public subnets** host the Application Load Balancer (ALB) and NAT Gateway.
* **Private subnets** contain sensitive resources such as the RDS database and EKS worker nodes.
* **Security groups** enforce least-privilege access, ensuring services only communicate where explicitly required.

This separation of concerns creates a secure perimeter, protecting the database and backend from direct internet exposure.

---

## 2. Amazon EKS (Elastic Kubernetes Service)

EKS is the compute backbone for NewsPulse. It allows containerized services — such as the API, processor, and frontend — to run with resilience and scalability.

* **Node groups** are divided between **system workloads** (monitoring, ingress controllers) and **application workloads** (API, processors).
* **IAM OIDC provider** is enabled to support IRSA (IAM Roles for Service Accounts), ensuring fine-grained, pod-level access to AWS services.

This approach future-proofs the system, enabling GitOps, automated scaling, and canary deployments.

---

## 3. Amazon RDS (Postgres)

News metadata, user profiles, and recommendation data are stored in **Amazon RDS (PostgreSQL)**.

* Fully managed service ensures **automated backups, patching, and failover**.
* Hosted in **private subnets** for security.
* Credentials are stored in **AWS Secrets Manager**, preventing hardcoding in code or configs.

The relational structure allows complex queries (e.g., “top articles in the last 24 hours per region”) with high efficiency.

---

## 4. Amazon S3 Buckets

Unstructured and analytical data flows into Amazon S3.

* **`news-raw` bucket**: Stores full articles and original content for archival and potential reprocessing.
* **`news-analytics` bucket**: Holds processed results, query logs, and Athena-compatible datasets.
* **Lifecycle policies** move older data to **Glacier**, minimizing cost while retaining history.

This provides a highly durable, cost-optimized storage layer.

---

## 5. Amazon Kinesis Data Stream

Real-time ingestion is powered by **Kinesis Data Streams**.

* Articles fetched by Lambda (ingestor) are published to the stream.
* A **processor service** consumes from the stream, calling **AWS Comprehend** for sentiment and entity tagging.
* The stream provides **buffering and decoupling**, ensuring no data loss during traffic spikes (e.g., breaking news events).

This ensures resilience and smooth handling of high-volume data pipelines.

---

## 6. Amazon ECR (Elastic Container Registry)

ECR serves as the **container image repository** for the project.

* Holds Docker images for the **API**, **processor**, and **frontend**.
* Integrated with **Jenkins CI/CD**, enabling build, scan, and push workflows.
* Secure IAM policies ensure only authorized pipelines and EKS nodes can pull images.

This guarantees a clean, auditable supply chain for application deployment.

---

## 7. Route53 & ACM (Domain & Security)

To provide a professional, secure experience, Route53 and ACM are configured:

* **Route53** manages DNS for `newspulse.dev` and subdomains (`api.newspulse.dev`, `app.newspulse.dev`).
* **ACM (AWS Certificate Manager)** issues TLS certificates for end-to-end HTTPS encryption.

This combination ensures the platform is both accessible and trustworthy to users.

---

## Conclusion

Step 1 lays down the **infrastructure backbone** of NewsPulse. With networking, compute, storage, and streaming in place, the system has the essential building blocks for scalability, reliability, and security. Subsequent steps will build on this foundation — deploying ingestion pipelines, processors, APIs, and frontend applications with confidence.

---




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

## ✅ Step 2: Install and Configure EKS Add-ons

Once the VPC, EKS, RDS, S3, Kinesis, and other resources are provisioned, the cluster itself needs its **core operational tools**:

1. **ArgoCD (GitOps Engine)**
<img width="1818" height="1047" alt="Screenshot from 2025-08-22 11-39-45" src="https://github.com/user-attachments/assets/198f1fbd-e066-48b4-9796-696ab3b9373a" />


   * Manage application deployments declaratively.
   * Sync Helm/Kustomize manifests from Git → EKS automatically.

3. **External Secrets Operator (ESO)**

   * Sync secrets from **AWS Secrets Manager** into Kubernetes.
   * Ensures no hardcoded secrets inside manifests.

4. **Ingress Controller (AWS Load Balancer Controller)**

   * Needed for routing traffic from ALB → Kubernetes services.
   * Works with Route53 + ACM certificates (from Step 1).

5. **Prometheus + Grafana + Alertmanager**

   * **Prometheus**: Collects cluster & app metrics.
   * **Grafana**: Visualizes dashboards (API latency, ingestion lag, CPU usage).
   * **Alertmanager**: Notifies on incidents (e.g., Slack, email, OpsGenie).

6. **OpenTelemetry Agent**

   * Collect traces/logs from services.
   * Helps debug latency across ingestion → processing → API chain.

---

## 🎯 End of Step 2 Goals

* GitOps fully functional → any manifest update in Git automatically applies to EKS.
* Secrets flow securely from AWS Secrets Manager to pods.
* Monitoring dashboards visible in Grafana.
* Ingress works → test domain resolves to EKS service.


---

# Step 2 — EKS Add-ons & Observability (Hands-on)

> Replace the placeholders:
>
> * `CLUSTER_NAME`, `AWS_REGION`, `ACCOUNT_ID`
> * `DOMAIN= new spulse.dev` (example)
> * `CERT_ARN` (ACM cert for `*.${DOMAIN}` in the same region as your ALB)

## 0) Prereqs

```bash
aws eks update-kubeconfig --name $CLUSTER_NAME --region $AWS_REGION
kubectl get nodes
kubectl create namespace ops || true
kubectl create namespace platform || true
kubectl create namespace monitoring || true
kubectl create namespace argocd || true
```

---

## 1) ArgoCD (GitOps)

### Install (Helm)

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm upgrade --install argocd argo/argo-cd \
  --namespace argocd \
  --set server.extraArgs="{--insecure}" \
  --set configs.params."server\\.insecure"=true
```

### Get initial admin password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
```

### Expose ArgoCD via ALB (Ingress)

> Requires AWS Load Balancer Controller (section 3). If not installed yet, skip and come back.

```yaml
# argocd/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd
  namespace: argocd
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80},{"HTTPS":443}]'
    alb.ingress.kubernetes.io/certificate-arn: CERT_ARN
    alb.ingress.kubernetes.io/ssl-redirect: '443'
spec:
  rules:
  - host: argocd.${DOMAIN}
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port: 
              number: 80
```

```bash
kubectl apply -f argocd/ingress.yaml
```

### App-of-Apps (GitOps root)

Create a **root** application so ArgoCD manages everything else from your Git repo.

```yaml
# argocd/root-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: newspulse-root
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<your-org>/newspulse.git
    targetRevision: main
    path: ops/apps
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

```bash
kubectl apply -f argocd/root-app.yaml
```

**Repo suggestion**

```
ops/
  apps/                # App-of-apps kustomization or App manifests
  base/                # Helm charts / Kustomize bases
  overlays/{dev,stg,prod}/
```

---

## 2) External Secrets Operator (Sync from AWS Secrets Manager)

### Install ESO

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm repo update
helm upgrade --install external-secrets external-secrets/external-secrets \
  --namespace platform --create-namespace
```

### IRSA for ESO (allow read from Secrets Manager)

Create an IAM policy that lets ESO read specific secrets (least privilege). Example inline policy name: `ESOReadSecrets`.

```bash
cat > es-irsa-policy.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "secretsmanager:GetSecretValue",
      "secretsmanager:DescribeSecret",
      "secretsmanager:ListSecretVersionIds"
    ],
    "Resource": [
      "arn:aws:secretsmanager:*:*:secret:newspulse/*"
    ]
  }]
}
EOF

aws iam create-policy --policy-name ESOReadSecrets \
  --policy-document file://es-irsa-policy.json
```

Create service account with IRSA (adjust OIDC and policy ARN):

```bash
OIDC_PROVIDER=$(aws eks describe-cluster --name $CLUSTER_NAME --region $AWS_REGION \
  --query "cluster.identity.oidc.issuer" --output text | sed -e 's#https://##')

helm upgrade --install eso-aws external-secrets/external-secrets \
  --namespace platform \
  --set installCRDs=true \
  --set serviceAccount.create=true \
  --set serviceAccount.name=eso-sa \
  --set "serviceAccount.annotations.eks\.amazonaws\.com/role-arn=arn:aws:iam::$ACCOUNT_ID:role/eso-irsa-role"
```

Create the role and attach the policy:

```bash
cat > trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Federated": "arn:aws:iam::$ACCOUNT_ID:oidc-provider/$OIDC_PROVIDER"},
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "$OIDC_PROVIDER:sub": "system:serviceaccount:platform:eso-sa"
      }
    }
  }]
}
EOF

aws iam create-role --role-name eso-irsa-role --assume-role-policy-document file://trust-policy.json
aws iam attach-role-policy --role-name eso-irsa-role --policy-arn arn:aws:iam::$ACCOUNT_ID:policy/ESOReadSecrets
```

### External Secret example (pull DB creds)

```yaml
# platform/db-secret.yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: rds-app-creds
  namespace: platform
spec:
  refreshInterval: 1h
  secretStoreRef:
    kind: ClusterSecretStore
    name: aws-secrets
  target:
    name: rds-app
    creationPolicy: Owner
  data:
  - secretKey: username
    remoteRef:
      key: newspulse/rds/app
      property: username
  - secretKey: password
    remoteRef:
      key: newspulse/rds/app
      property: password
```

Create the store (to AWS SM):

```yaml
# platform/clustersecretstore.yaml
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: aws-secrets
spec:
  provider:
    aws:
      service: SecretsManager
      region: AWS_REGION
      auth:
        jwt:
          serviceAccountRef:
            name: eso-sa
            namespace: platform
```

```bash
kubectl apply -f platform/clustersecretstore.yaml
kubectl apply -f platform/db-secret.yaml
kubectl -n platform get secret rds-app -o yaml
```

---

## 3) AWS Load Balancer Controller (Ingress → ALB)

### IRSA + Install

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update

# IAM policy
curl -o alb-iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json
aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://alb-iam-policy.json

# Service account w/ IRSA
kubectl create namespace kube-system --dry-run=client -o yaml | kubectl apply -f -
eksctl create iamserviceaccount \
  --name aws-load-balancer-controller \
  --namespace kube-system \
  --cluster $CLUSTER_NAME \
  --attach-policy-arn arn:aws:iam::$ACCOUNT_ID:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve \
  --region $AWS_REGION

# Install chart
helm upgrade --install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=$CLUSTER_NAME \
  --set region=$AWS_REGION \
  --set vpcId=$(aws eks describe-cluster --name $CLUSTER_NAME --region $AWS_REGION --query "cluster.resourcesVpcConfig.vpcId" --output text) \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

**Health check**

```bash
kubectl -n kube-system rollout status deploy/aws-load-balancer-controller
```

---

## 4) Ingress test (Hello service)

```yaml
# ops/base/hello/deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello
  namespace: ops
spec:
  replicas: 2
  selector: { matchLabels: { app: hello } }
  template:
    metadata: { labels: { app: hello } }
    spec:
      containers:
      - name: hello
        image: public.ecr.aws/nginx/nginx:latest
        ports: [{ containerPort: 80 }]
---
apiVersion: v1
kind: Service
metadata:
  name: hello
  namespace: ops
spec:
  selector: { app: hello }
  ports: [{ port: 80, targetPort: 80 }]
```

```yaml
# ops/base/hello/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello
  namespace: ops
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/certificate-arn: CERT_ARN
    alb.ingress.kubernetes.io/ssl-redirect: '443'
spec:
  rules:
  - host: hello.${DOMAIN}
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: hello
            port:
              number: 80
```

```bash
kubectl apply -f ops/base/hello/deploy.yaml
kubectl apply -f ops/base/hello/ingress.yaml
# After ALB provision, create DNS record in Route53 to the ALB hostname → hello.${DOMAIN}
```

---

## 5) Monitoring — kube-prometheus-stack (Prometheus, Grafana, Alertmanager)

### Install

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm upgrade --install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set grafana.adminPassword='Admin123!' \
  --set grafana.ingress.enabled=true \
  --set grafana.ingress.ingressClassName=alb \
  --set grafana.ingress.annotations."alb\.ingress\.kubernetes\.io/scheme"=internet-facing \
  --set grafana.ingress.annotations."alb\.ingress\.kubernetes\.io/target-type"=ip \
  --set grafana.ingress.annotations."alb\.ingress\.kubernetes\.io/listen-ports"='[{"HTTP":80},{"HTTPS":443}]' \
  --set grafana.ingress.annotations."alb\.ingress\.kubernetes\.io/certificate-arn"=CERT_ARN \
  --set grafana.ingress.annotations."alb\.ingress\.kubernetes\.io/ssl-redirect"='443' \
  --set grafana.ingress.hosts="{grafana.${DOMAIN}}" \
  --set prometheus.ingress.enabled=true \
  --set prometheus.ingress.ingressClassName=alb \
  --set prometheus.ingress.annotations."alb\.ingress\.kubernetes\.io/scheme"=internet-facing \
  --set prometheus.ingress.annotations."alb\.ingress\.kubernetes\.io/target-type"=ip \
  --set prometheus.ingress.annotations."alb\.ingress\.kubernetes\.io/listen-ports"='[{"HTTP":80},{"HTTPS":443}]' \
  --set prometheus.ingress.annotations."alb\.ingress\.kubernetes\.io/certificate-arn"=CERT_ARN \
  --set prometheus.ingress.annotations."alb\.ingress\.kubernetes\.io/ssl-redirect"='443' \
  --set prometheus.ingress.hosts="{prometheus.${DOMAIN}}"
```

### Dashboards to import

* **Kubernetes / Compute Resources / Namespace, Pod**
* **Node Exporter / Nodes**
* **NGINX or ALB** (if you export metrics)
* Later: custom dashboards for **API latency**, **Kinesis lag**, **processor throughput**.

---

## 6) OpenTelemetry Collector (Traces/Logs)

```yaml
# monitoring/otel-collector.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-collector-config
  namespace: monitoring
data:
  otel-collector-config.yaml: |
    receivers:
      otlp:
        protocols: { http: {}, grpc: {} }
    exporters:
      logging: {}
    service:
      pipelines:
        traces:
          receivers: [otlp]
          exporters: [logging]
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: otel-collector
  namespace: monitoring
spec:
  replicas: 1
  selector: { matchLabels: { app: otel-collector } }
  template:
    metadata: { labels: { app: otel-collector } }
    spec:
      containers:
      - name: otel-collector
        image: otel/opentelemetry-collector:latest
        args: ["--config=/conf/otel-collector-config.yaml"]
        volumeMounts:
          - name: config
            mountPath: /conf
      volumes:
        - name: config
          configMap:
            name: otel-collector-config
            items:
              - key: otel-collector-config.yaml
                path: otel-collector-config.yaml
```

```bash
kubectl apply -f monitoring/otel-collector.yaml
```

Update your API/processor later to export OTLP → `otel-collector.monitoring:4317`.

---

## 7) Alerting (quick start)

Example: **High 5xx on ingress service** (PrometheusRule):

```yaml
# monitoring/alerts-5xx.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: ingress-5xx
  namespace: monitoring
spec:
  groups:
  - name: ingress.rules
    rules:
    - alert: HighIngress5xx
      expr: sum(rate(nginx_ingress_controller_requests{status=~"5.."}[5m])) > 5
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "High 5xx rate on Ingress"
        description: "Ingress 5xx rate > 5 rps for 10m"
```

```bash
kubectl apply -f monitoring/alerts-5xx.yaml
```

(Configure Alertmanager to route to Slack/Email later.)

---

## 8) Validation Checklist

```bash
# ArgoCD
kubectl -n argocd get pods
# ESO secret synced
kubectl -n platform get externalsecret,secrets
# ALB controller healthy
kubectl -n kube-system get deploy aws-load-balancer-controller
# Ingress working
kubectl -n ops get ingress
# Monitoring stack
kubectl -n monitoring get pods,ingress
```

* Visit:

  * `https://argocd.${DOMAIN}`
  * `https://grafana.${DOMAIN}`
  * `https://hello.${DOMAIN}`

---

## 9) Production Notes

* **Network policies**: add Calico/Cilium and restrict namespace egress where possible.
* **WAF**: attach AWS WAF to the ALB for API protection (SQLi/XSS managed rules).
* **TLS**: use your ACM wildcard, rotate yearly; consider cert-manager if you ever move to non-ALB ingress.
* **RBAC**: read-only roles for Grafana/ArgoCD viewers.
* **Backups**: Prometheus snapshots, Grafana dashboards exported to Git.

---





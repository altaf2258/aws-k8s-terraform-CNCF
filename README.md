# AWS Kubernetes DevOps Platform

A production-style Kubernetes platform built on AWS using **Terraform, kubeadm, Calico, NGINX Ingress, AWS ALB, and Amazon RDS**.

The project demonstrates how to provision cloud infrastructure with Infrastructure as Code, build a Kubernetes cluster, deploy a frontend/backend application, expose services through an AWS Application Load Balancer and NGINX Ingress, and implement Kubernetes production-hardening features such as probes, resource limits, PDB and HPA.

---

## 🏗️ Architecture

```text
                         Internet
                            |
                     Cloudflare DNS
                            |
                            v
                AWS Application Load Balancer
                         HTTP :80
                            |
                     Target Group
                            |
                   Kubernetes NodePort
                         :30911
                            |
                            v
                 NGINX Ingress Controller
                            |
              +-------------+-------------+
              |                           |
              v                           v
     management.webanix...       apimanagement.webanix...
              |                           |
              v                           v
      Frontend Service             Backend Service
          :3000                         :4000
              |                           |
              v                           v
       Frontend Pods              Backend Pods
           :80                         :3005
                                          |
                                          v
                                  Amazon RDS MySQL
                                      :3306
```

---

## 🚀 Technology Stack

| Category                | Technology                    |
| ----------------------- | ----------------------------- |
| Cloud                   | AWS                           |
| Infrastructure as Code  | Terraform                     |
| Container Orchestration | Kubernetes                    |
| Kubernetes Bootstrap    | kubeadm                       |
| CNI                     | Calico                        |
| Operating System        | Ubuntu 24.04                  |
| Container Runtime       | containerd                    |
| Ingress                 | NGINX Ingress Controller      |
| Load Balancer           | AWS Application Load Balancer |
| Database                | Amazon RDS MySQL 8.0          |
| DNS                     | Cloudflare                    |
| Autoscaling             | Kubernetes HPA                |
| Availability            | PodDisruptionBudget           |
| Monitoring Metrics      | Metrics Server                |
| Container Images        | Docker                        |
| Region                  | AWS us-east-1                 |

---

## ☁️ AWS Infrastructure

The infrastructure is provisioned using Terraform.

### VPC

```text
VPC: 10.0.0.0/16
```

### Public Subnets

```text
us-east-1a
10.0.1.0/24

us-east-1b
10.0.2.0/24
```

Used for the internet-facing Application Load Balancer.

### Private Application Subnets

```text
us-east-1a
10.0.11.0/24

us-east-1b
10.0.12.0/24
```

Used for Kubernetes EC2 nodes.

### Private Database Subnets

```text
us-east-1a
10.0.21.0/24

us-east-1b
10.0.22.0/24
```

Used for Amazon RDS.

### NAT Gateway

A NAT Gateway is used to provide outbound internet connectivity from private resources.

One NAT Gateway is currently used for the DEV environment to control cost.

---

## 🖥️ Kubernetes Cluster

The Kubernetes cluster is created using `kubeadm`.

| Node        | Role          | Private IP |
| ----------- | ------------- | ---------- |
| k8s-master  | Control Plane | 10.0.11.10 |
| k8s-worker1 | Worker        | 10.0.11.88 |
| k8s-worker2 | Worker        | 10.0.12.60 |

### Kubernetes Version

```text
v1.29.15
```

### Container Runtime

```text
containerd 2.3.6
```

### CNI

```text
Calico
```

---

## 📦 Application Architecture

The application consists of two main workloads.

### Frontend

```text
Deployment:
frontend-deployment

Replicas:
2

Container Port:
80

Service:
management-frontend-service

Service Port:
3000
```

### Backend

```text
Deployment:
management-backend

Replicas:
2

Container Port:
3005

Service:
management-backend-service

Service Port:
4000
```

### Application Images

```text
thedrdoom/management-frontend:latest

thedrdoom/management-backend:latest
```

---

## 🌐 Kubernetes Ingress

NGINX Ingress Controller is used for host-based routing.

### Frontend

```text
management.webanixsolutions.com
        |
        v
management-frontend-service:3000
```

### Backend

```text
apimanagement.webanixsolutions.com
        |
        v
management-backend-service:4000
```

Ingress Controller:

```text
NGINX Ingress Controller
v1.14.1
```

---

## ⚖️ AWS ALB

The AWS Application Load Balancer is provisioned using Terraform.

```text
Internet
   |
   v
AWS ALB :80
   |
   v
Target Group
   |
   v
Kubernetes Nodes :30911
```

### ALB Configuration

```text
Load Balancer:
Application Load Balancer

Listener:
HTTP :80

Target Type:
Instance

Target Port:
30911

Health Check:
HTTP /
```

The ALB forwards traffic to the Kubernetes NGINX Ingress NodePort.

---

## 🔐 Security

Security is implemented at multiple layers.

### Kubernetes Security Group

Important rules:

```text
SSH 22
Restricted to administrator IP

Kubernetes API 6443
Restricted to administrator IP

HTTP 80
Allowed as required

HTTPS 443
Allowed as required

NodePort 30911
Allowed from ALB Security Group
```

### RDS Security Group

MySQL traffic is restricted to Kubernetes nodes:

```text
TCP 3306
Source: Kubernetes Security Group
```

RDS is not publicly accessible.

### Network Design

```text
Internet
   |
Public ALB
   |
Private Kubernetes Nodes
   |
Private RDS
```

This reduces direct internet exposure of the application and database infrastructure.

---

## 🔑 Secrets Management

Database credentials are not hard-coded into the application Deployment.

Kubernetes uses:

```text
Secret:
rds-secret

Namespace:
app
```

The backend consumes the secret using:

```yaml
envFrom:
  - secretRef:
      name: rds-secret
```

AWS RDS credentials are managed through AWS Secrets Manager.

> Secrets and passwords are intentionally not stored in this repository or README.

---

## ❤️ Health Checks

Both frontend and backend Deployments use Kubernetes health probes.

### Frontend

```text
Liveness Probe:
TCP :80

Readiness Probe:
TCP :80
```

### Backend

```text
Liveness Probe:
TCP :3005

Readiness Probe:
TCP :3005
```

### Purpose

**Readiness Probe**

Determines whether a Pod is ready to receive traffic.

**Liveness Probe**

Determines whether Kubernetes should restart an unhealthy container.

---

## 📊 Resource Management

Application containers have CPU and memory requests/limits.

```yaml
requests:
  cpu: "100m"
  memory: "128Mi"

limits:
  cpu: "500m"
  memory: "512Mi"
```

### Why?

Requests help Kubernetes schedule Pods.

Limits prevent a container from consuming unlimited resources.

---

## 🔄 Rolling Updates

The application Deployments use:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0
    maxSurge: 1
```

This allows Pods to be replaced gradually during application updates.

The goal is to maintain application availability during deployments.

---

## 🛡️ PodDisruptionBudget

PDBs are configured for both frontend and backend.

```yaml
minAvailable: 1
```

This helps maintain at least one available Pod during voluntary disruptions such as node draining.

---

## 📈 Horizontal Pod Autoscaler

HPA is configured for both frontend and backend.

```text
Minimum replicas: 2
Maximum replicas: 5
CPU target: 60%
```

Example:

```text
management-backend-hpa
management-frontend-hpa
```

The HPA uses CPU utilization to automatically adjust the number of application Pods.

---

## 📡 Metrics Server

Metrics Server provides CPU and memory metrics for:

```bash
kubectl top pods
kubectl top nodes
```

It is also required for the configured HPA.

During setup, kubelet certificate validation failed because the kubelet certificates did not contain node IP addresses as IP SANs.

For this DEV cluster, Metrics Server was configured with:

```text
--kubelet-insecure-tls
```

This was used as a DEV workaround for the certificate configuration.

---

## 🏗️ Terraform Structure

```text
kubeadm-dev-infra/
│
├── environments/
│   └── dev/
│       ├── main.tf
│       ├── outputs.tf
│       ├── variables.tf
│       ├── frontend-deployment.yml
│       ├── terraform.tfstate
│       └── .terraform.lock.hcl
│
├── modules/
│   ├── alb/
│   ├── bastion/
│   ├── ec2/
│   ├── rds/
│   ├── security-groups/
│   └── vpc/
│
├── provider.tf
├── versions.tf
│
├── video-prod-back/
└── video-prod-front/
```

Terraform modules separate infrastructure responsibilities and make the configuration reusable.

---

## 🗄️ Terraform Remote State

Terraform state is stored remotely in Amazon S3.

```text
S3 Bucket:
kubeadm-dev-terraform-state-webanix

State Key:
dev/terraform.tfstate

Region:
us-east-1
```

S3 versioning is enabled.

Remote state provides centralized state management and protects the infrastructure workflow from relying only on local state.

---

## 🔧 Common Terraform Commands

Initialize:

```bash
terraform init
```

Validate:

```bash
terraform validate
```

Create execution plan:

```bash
terraform plan
```

Apply infrastructure:

```bash
terraform apply
```

Show outputs:

```bash
terraform output
```

---

## ☸️ Common Kubernetes Commands

Check cluster:

```bash
kubectl get nodes -o wide
```

Check all Pods:

```bash
kubectl get pods -A
```

Check application Pods:

```bash
kubectl get pods -n app -o wide
```

Check Services:

```bash
kubectl get svc -n app
```

Check Ingress:

```bash
kubectl get ingress -n app
```

Check HPA:

```bash
kubectl get hpa -n app
```

Check PDB:

```bash
kubectl get pdb -n app
```

Check resource metrics:

```bash
kubectl top pods -n app
kubectl top nodes
```

Check logs:

```bash
kubectl logs <pod-name> -n app
```

Describe a Pod:

```bash
kubectl describe pod <pod-name> -n app
```

---

## 🔄 Rollout & Rollback

Check rollout:

```bash
kubectl rollout status deployment/management-backend -n app
```

View rollout history:

```bash
kubectl rollout history deployment/management-backend -n app
```

Rollback:

```bash
kubectl rollout undo deployment/management-backend -n app
```

For production CI/CD, immutable image tags should be preferred over `latest` so every release can be traced and rolled back reliably.

---

## 🧪 Validation

The following validations were completed:

```text
✓ 3 Kubernetes nodes Ready
✓ Calico CNI healthy
✓ Frontend Deployment 2/2 Ready
✓ Backend Deployment 2/2 Ready
✓ Frontend Service routing
✓ Backend Service routing
✓ NGINX Ingress routing
✓ AWS ALB routing
✓ Cloudflare DNS resolution
✓ RDS connectivity
✓ Resource requests/limits
✓ Liveness probes
✓ Readiness probes
✓ RollingUpdate strategy
✓ PodDisruptionBudget
✓ Metrics Server
✓ HPA
```

---

## 🐛 Troubleshooting Examples

### Metrics Server TLS Error

Problem:

```text
x509: cannot validate certificate for <node-ip>
because it doesn't contain any IP SANs
```

Resolution for DEV:

```text
--kubelet-insecure-tls
```

---

### Kubernetes Service Test

Backend Service was tested internally:

```bash
curl http://management-backend-service:4000
```

The application returned:

```text
Cannot GET /
```

This confirmed that the Service was successfully reaching the backend application, while `/` itself was not an application route.

---

### API Route

The API hostname reached the backend, but a `/signIn` request returned an application-level `404`.

This was treated as an application/code-side routing issue and was not modified from the infrastructure side.

---

## 📋 Current Project Status

| Component                 | Status       |
| ------------------------- | ------------ |
| AWS VPC                   | ✅ Completed  |
| Public/Private Subnets    | ✅ Completed  |
| NAT Gateway               | ✅ Completed  |
| Security Groups           | ✅ Completed  |
| Kubernetes EC2 Nodes      | ✅ Completed  |
| RDS MySQL                 | ✅ Completed  |
| Terraform Remote State    | ✅ Completed  |
| kubeadm Cluster           | ✅ Completed  |
| Calico                    | ✅ Completed  |
| Frontend                  | ✅ Completed  |
| Backend                   | ✅ Completed  |
| Kubernetes Services       | ✅ Completed  |
| NGINX Ingress             | ✅ Completed  |
| AWS ALB                   | ✅ Completed  |
| Cloudflare DNS            | ✅ Configured |
| Resource Requests/Limits  | ✅ Completed  |
| Liveness/Readiness Probes | ✅ Completed  |
| Rolling Updates           | ✅ Completed  |
| PDB                       | ✅ Completed  |
| Metrics Server            | ✅ Completed  |
| HPA                       | ✅ Completed  |
| CI/CD                     | ⏳ Planned    |
| HTTPS/ACM                 | ⏳ Planned    |
| Pod Anti-Affinity         | ⏳ Planned    |
| Monitoring/Alerting       | ⏳ Planned    |
| GitOps                    | ⏳ Planned    |

---

## 🔮 Future Improvements

### 1. CI/CD

Planned pipeline:

```text
Git Push
   |
Bitbucket
   |
Build
   |
Unit Tests
   |
Trivy Security Scan
   |
Docker Image Build
   |
Docker Registry
   |
Kubernetes Deployment
   |
Rolling Update
```

---

### 2. HTTPS

Planned implementation:

```text
Cloudflare
    |
HTTPS
    |
AWS ALB :443
    |
NGINX Ingress
```

Planned components:

* AWS ACM certificate
* ALB HTTPS listener
* HTTP → HTTPS redirect
* Cloudflare SSL configuration

---

### 3. Pod Anti-Affinity

Pod anti-affinity will be considered in a later phase to distribute frontend/backend replicas across different Kubernetes nodes.

---

### 4. Monitoring

Future monitoring can include:

```text
Prometheus
Grafana
Alertmanager
```

or an AWS-native monitoring solution.

---

### 5. GitOps

A future GitOps implementation can use:

```text
Git
 |
Argo CD
 |
Kubernetes
```

---

## 🎯 DevOps Concepts Demonstrated

This project demonstrates practical knowledge of:

* Infrastructure as Code
* Terraform modules
* Terraform remote state
* AWS VPC architecture
* Public/private subnet design
* NAT Gateway
* Security Groups
* EC2
* RDS
* Application Load Balancer
* DNS
* Kubernetes architecture
* kubeadm
* Calico CNI
* Deployments
* ReplicaSets
* Services
* NodePort
* Ingress
* Resource requests and limits
* Liveness and readiness probes
* Rolling deployments
* Rollbacks
* PodDisruptionBudget
* Metrics Server
* Horizontal Pod Autoscaler
* Kubernetes Secrets
* Application troubleshooting
* Cloud security fundamentals

---

## 💼 Interview Explanation

> "I built a production-style Kubernetes platform on AWS using Terraform and kubeadm. Terraform provisions the VPC, private Kubernetes nodes, RDS, security groups and ALB. I bootstrapped a three-node Kubernetes cluster using kubeadm with Calico as the CNI. The frontend and backend are deployed as separate Kubernetes Deployments and Services. External traffic comes through Cloudflare and an AWS ALB, which forwards traffic to the NGINX Ingress Controller through NodePort. I also implemented resource requests and limits, readiness and liveness probes, rolling updates, PDB and HPA. CI/CD, HTTPS, monitoring and GitOps are planned for the next phase."

---

## 📌 Project Goals

The main goals of this project are:

1. Build AWS infrastructure using Terraform.
2. Deploy a Kubernetes cluster using kubeadm.
3. Deploy a containerized frontend/backend application.
4. Implement secure network communication between application and database layers.
5. Implement external traffic routing using ALB and NGINX Ingress.
6. Implement Kubernetes production-hardening practices.
7. Demonstrate scalability using HPA.
8. Build a foundation for future CI/CD, monitoring and GitOps implementation.

---

## 👨‍💻 Project Focus

**AWS | Terraform | Kubernetes | kubeadm | Docker | Calico | NGINX Ingress | ALB | RDS | Cloudflare | HPA | PDB | DevOps**

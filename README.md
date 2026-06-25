# KaaS on OpenStack — Kubernetes as a Service with Cluster API

> A production-grade self-service Kubernetes platform built on OpenStack using Cluster API (CAPI) + CAPO.
> One `kubectl apply` = one new Kubernetes cluster provisioned automatically on OpenStack.

## Stack

| Tool | Role |
|---|---|
| OpenStack (ICOSNET) | IaaS provider |
| Cluster API (CAPI) | Declarative cluster lifecycle management |
| CAPO | OpenStack infrastructure provider for CAPI |
| kind | Management cluster bootstrap |
| Terraform | Infrastructure provisioning as code |
| Helm | Application deployment (Prometheus, Grafana, ArgoCD) |
| Prometheus + Grafana | Observability stack |
| ArgoCD | GitOps cluster lifecycle |

## Screenshots

![Tools installed](screenshots/02-all-tools-installed.png)

## Author

**Zekkour Abderraouf** — Final year Networks, Systems & Telecommunications — ENSTICP Algiers
- GitHub: [@Abderraoufzekkour](https://github.com/Abderraoufzekkour)
- LinkedIn: [zekkour-abderraouf](https://www.linkedin.com/in/zekkour-abderraouf)

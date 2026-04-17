# Checklist Bàn Giao Hệ Thống DevOps/Infra

## 1. Hệ thống chung

| # | Hệ thống | URL / Repo | Mô tả | Account truy cập | Ghi chú |
|---|----------|-----------|-------|-------------------|---------|
| 1 | Gitlab Server | https://gitlab.proteam.click/ | Source code management | | |
| 2 | Jenkins | https://jenkin.proteam.click/ | CI/CD automation server | | |
| 3 | Harbor | https://harbor.proteam.click/ | Container image registry | | |

## 2. Monitorings (ECOM)

| # | Thành phần | URL / Repo | Mô tả | Account truy cập | Ghi chú |
|---|-----------|-----------|-------|-------------------|---------|
| 1 | Ansible Setup Monitorings | https://gitlab.proteam.click/monitorings/ansible-setup-monitorings.git | Ansible playbook cài đặt Prometheus + Grafana + Alertmanager + Exporters | | Hiện tại mới setup cho ECOM |
| 2 | Prometheus | | Metrics collection & alerting | | |
| 3 | Grafana | | Dashboard & visualization | | |
| 4 | Alertmanager | | Alert routing (email) | | |

## 3. Kubernetes

| # | Thành phần | Repo / URL | Mô tả | Account truy cập | Ghi chú |
|---|-----------|-----------|-------|-------------------|---------|
| 1 | ArgoCD Application | gitops-argocd/sts-cluster-gitops | Repo ArgoCD application để tạo K8s cluster | | |
| 2 | Helm Values | sts-helm-values | Config values cho các apps trên K8s | | |
| 3 | Infra Terraform | sts-infra-terraform | Khởi tạo các resource AWS liên quan cho K8s | | |

## 4. CI/CD

| # | Thành phần | Repo / URL | Mô tả | Account truy cập | Ghi chú |
|---|-----------|-----------|-------|-------------------|---------|
| 1 | Jenkins Seed Jobs | jenkins-seed-jobs | Job DSL tạo Jenkins jobs tự động | | |
| 2 | Jenkins Shared Library | sts-jenkins-shared-library | Shared library dùng chung cho Jenkins pipelines | | |

## 5. AWS Account

| # | Thành phần | Account ID / URL | Mô tả | Account truy cập | Ghi chú |
|---|-----------|-----------------|-------|-------------------|---------|
| 1 | AWS Console | | | | |
| 2 | IAM User / Role | | | | |

## 6. Thông tin bổ sung

| # | Hạng mục | Chi tiết | Ghi chú |
|---|---------|---------|---------|
| 1 | VPN | | |
| 2 | SSH Key | | |
| 3 | Domain / DNS | | |
| 4 | SSL Certificate | | |
| 5 | Backup | | |

---

> **Người bàn giao:**  
> **Người nhận bàn giao:**  
> **Ngày bàn giao:**  

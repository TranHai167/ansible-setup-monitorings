# Hướng dẫn cài đặt & vận hành hệ thống Monitoring

## Mục lục

1. [Tổng quan hệ thống](#1-tổng-quan-hệ-thống)
2. [Yêu cầu hạ tầng & mạng](#2-yêu-cầu-hạ-tầng--mạng)
3. [Cài đặt môi trường trên máy local](#3-cài-đặt-môi-trường-trên-máy-local)
4. [Clone & cấu hình project](#4-clone--cấu-hình-project)
5. [Deploy monitoring stack](#5-deploy-monitoring-stack)
6. [Cấu hình Grafana](#6-cấu-hình-grafana)
7. [Thêm target EC2 mới](#7-thêm-target-ec2-mới)
8. [Hướng dẫn chọn role phù hợp](#8-hướng-dẫn-chọn-role-phù-hợp)
9. [Monitor liên AWS Account](#9-monitor-liên-aws-account)
10. [Vận hành & bảo trì](#10-vận-hành--bảo-trì)
11. [Troubleshooting](#11-troubleshooting)

---

## 1. Tổng quan hệ thống

### Kiến trúc

```
                         ┌─────────────────────────────────────┐
                         │     EC2 Monitoring Server            │
                         │  ┌─────────────┐  ┌──────────────┐  │
  Grafana Dashboard ◄────│──│  Prometheus  │  │ Alertmanager │  │
  http://18.138.145.51:3000 │  (:9090)     │  │   (:9093)    │  │
                         │  └──────┬──────┘  └──────────────┘  │
                         └─────────┼───────────────────────────┘
                                   │ scrape (pull metrics)
                    ┌──────────────┼──────────────┐
                    ▼              ▼               ▼
           ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
           │  EC2 Target A │ │  EC2 Target B │ │  EC2 Target C │
           │               │ │               │ │               │
           │ node_exporter │ │ node_exporter │ │ node_exporter │
           │ cadvisor      │ │ phpfpm_export │ │ docker_export │
           │   (:9100)     │ │ cadvisor      │ │ cron_monitor  │
           │   (:8080)     │ │  (:9100)      │ │   (:9100)     │
           └──────────────┘ │  (:8080)      │ └──────────────┘
                            │  (:9253)      │
                            └──────────────┘
```

### Thành phần

| Thành phần | Vai trò | Chạy trên |
|------------|---------|-----------|
| **Prometheus** | Thu thập & lưu trữ metrics | EC2 Monitoring |
| **Grafana** | Dashboard hiển thị | EC2 Monitoring |
| **Alertmanager** | Gửi alert qua email | EC2 Monitoring |
| **node_exporter** | Metrics hệ thống (CPU, RAM, Disk) | EC2 Target |
| **cAdvisor** | Metrics Docker container | EC2 Target |
| **docker_exporter** | Thay thế cAdvisor (cho ARM/overlayfs) | EC2 Target |
| **phpfpm_exporter** | Metrics PHP-FPM | EC2 Target |
| **magento_cron_monitor** | Metrics Magento cron jobs | EC2 Target |

### Link truy cập

| Service | URL |
|---------|-----|
| Grafana Dashboard | http://18.138.145.51:3000/dashboards |
| Prometheus | http://18.138.145.51:9090 |
| Alertmanager | http://18.138.145.51:9093 |
| Git Repo | https://gitlab.proteam.click/monitorings/ansible-setup-monitorings.git |

---

## 2. Yêu cầu hạ tầng & mạng

### EC2 Monitoring Server

- OS: Ubuntu hoặc Amazon Linux 2023
- Docker + Docker Compose
- Ports mở: 9090 (Prometheus), 3000 (Grafana), 9093 (Alertmanager)

### EC2 Target Servers

- OS: Ubuntu (Debian-based)
- Docker (nếu cần monitor containers)
- SSM Agent đã cài và active

### ⚠️ Yêu cầu mạng (QUAN TRỌNG)

Prometheus pull metrics từ target qua **private IP**. Các EC2 phải thông mạng với nhau:

```
EC2 Monitoring ──(private network)──► EC2 Target
                                      port 9100 (node_exporter)
                                      port 8080 (cAdvisor)
                                      port 9253 (phpfpm_exporter)
```

#### Cùng VPC
- Đảm bảo Security Group của target cho phép inbound từ Monitoring SG

#### Khác VPC (cùng account hoặc khác account)
- Cần **Transit Gateway** để kết nối các VPC (thiết kế hub-and-spoke, hỗ trợ multi-account)
- Route table mỗi VPC phải có route đến CIDR của các VPC khác qua Transit Gateway
- Security Group target cho phép inbound từ CIDR của VPC monitoring

#### Security Group Rules cho Target EC2

| Port | Protocol | Source | Mô tả |
|------|----------|--------|-------|
| 9100 | TCP | Monitoring SG / CIDR | node_exporter |
| 8080 | TCP | Monitoring SG / CIDR | cAdvisor |
| 9253 | TCP | Monitoring SG / CIDR | phpfpm_exporter |

#### Security Group Rules cho Monitoring EC2

| Port | Protocol | Source | Mô tả |
|------|----------|--------|-------|
| 9090 | TCP | Your IP / VPN | Prometheus UI |
| 3000 | TCP | ALB SG / Your IP | Grafana |
| 9093 | TCP | Your IP / VPN | Alertmanager |

#### Kiểm tra kết nối

Từ EC2 Monitoring, test kết nối đến target:

```bash
# Test node_exporter
curl -s http://<TARGET_PRIVATE_IP>:9100/metrics | head -5

# Test cAdvisor
curl -s http://<TARGET_PRIVATE_IP>:8080/metrics | head -5
```

Nếu timeout → kiểm tra Security Group, Route Table, Transit Gateway attachment.

---

## 3. Cài đặt môi trường trên máy local

Ansible chạy từ máy local (Ubuntu WSL hoặc Linux), kết nối đến EC2 qua **AWS SSM** (không cần SSH key).

### 3.1 Cài Ansible

```bash
sudo apt update
sudo apt install python3-pip python3-boto3 -y
pip install ansible
```

### 3.2 Cài AWS SSM Plugin

```bash
# SSM connection plugin cho Ansible
ansible-galaxy collection install community.aws

# AWS Session Manager Plugin
curl "https://s3.amazonaws.com/session-manager-downloads/plugin/latest/ubuntu_64bit/session-manager-plugin.deb" \
  -o /tmp/session-manager-plugin.deb
sudo dpkg -i /tmp/session-manager-plugin.deb
rm /tmp/session-manager-plugin.deb
```

### 3.3 Cấu hình AWS CLI

```bash
# Cài AWS CLI (nếu chưa có)
pip install awscli

# Cấu hình credentials
aws configure
# Hoặc dùng AWS Profile
aws configure --profile <profile-name>
```

Cần quyền IAM:
- `ssm:StartSession`
- `ssm:TerminateSession`
- `ssm:SendCommand`
- `s3:PutObject` / `s3:GetObject` (cho SSM bucket)

### 3.4 Verify

```bash
ansible --version
python3 -c "import boto3; print(boto3.__version__)"
ansible-galaxy collection list | grep community.aws
session-manager-plugin --version
aws sts get-caller-identity
```

---

## 4. Clone & cấu hình project

### 4.1 Clone repo

```bash
git clone https://gitlab.proteam.click/monitorings/ansible-setup-monitorings.git
cd ansible-setup-monitorings
```

### 4.2 Cấu hình biến môi trường

```bash
cp .env.example .env
```

Sửa file `.env`:

```bash
# SMTP cho alert email
export SMTP_HOST=smtp.office365.com
export SMTP_PORT=587
export SMTP_USER=no_reply_test@sts.vn
export SMTP_PASSWORD=<password>

# Email nhận alert
export FROM_EMAIL=no_reply_test@sts.vn
export TO_EMAIL="haitn2@sts.vn, cuongph@sts.vn"

# Magento DB (chỉ cần nếu dùng magento_cron_monitor)
export MAGENTO_DB_HOST=<rds-endpoint>
export MAGENTO_DB_PORT=3306
export MAGENTO_DB_USER=<db-user>
export MAGENTO_DB_PASSWORD=<db-password>
export MAGENTO_DB_NAME=openasia
```

### 4.3 Chọn inventory file

Project có 2 inventory:
- `inventory/production.yml` — môi trường Production
- `inventory/uat.yml` — môi trường UAT

Mỗi inventory chứa:
- `monitoring_server` — EC2 chạy Prometheus/Grafana
- `target_servers` — các EC2 cần monitor

### 4.4 Lưu ý WSL

Thư mục `/mnt/c/...` là world-writable nên Ansible bỏ qua `ansible.cfg`. Luôn thêm prefix:

```bash
export ANSIBLE_CONFIG=./ansible.cfg
# Hoặc thêm trước mỗi lệnh
ANSIBLE_CONFIG=./ansible.cfg ansible-playbook ...
```

---

## 5. Deploy monitoring stack

### Bước 1: Deploy exporters lên tất cả target EC2

```bash
source .env
ANSIBLE_CONFIG=./ansible.cfg ansible-playbook playbooks/setup-target.yml -i inventory/production.yml
```

### Bước 2: Deploy Prometheus + Grafana + Alertmanager

```bash
source .env
ANSIBLE_CONFIG=./ansible.cfg ansible-playbook playbooks/setup-monitoring.yml -i inventory/production.yml
```

### Bước 3: Verify

Trên target EC2:

```bash
# Kiểm tra exporters
sudo ss -tlnp | grep -E ':(9100|8080|9253)'
curl -s http://localhost:9100/metrics | head -5
```

Trên monitoring EC2:

```bash
docker ps   # Phải thấy prometheus, grafana, alertmanager
curl -s http://localhost:9090/-/healthy
```

Prometheus Targets: http://18.138.145.51:9090/targets — tất cả targets phải UP.

---

## 6. Cấu hình Grafana

### 6.1 Đăng nhập

- URL: http://18.138.145.51:3000
- User: `admin` / Password: `ecom_admin@2026` (đổi sau lần đầu)

### 6.2 Thêm Data Source

1. Home → Administration → Data Sources → Add data source
2. Chọn **Prometheus**
3. URL: `http://prometheus:9090` (dùng Docker internal DNS)
4. Click **Save & Test**

### 6.3 Import Dashboards

| Dashboard | Cách import | Dùng cho |
|-----------|-------------|----------|
| Node Exporter Full | Import ID: `1860` | Host metrics (CPU, RAM, Disk) |
| cAdvisor Docker | Import ID: `14282` | Docker container metrics |
| PHP-FPM | Import ID: `4912` | PHP-FPM metrics |
| Magento Cron | Upload file `grafana/magento-cron-dashboard.json` | Magento cron jobs |

Import: Dashboards → New → Import → nhập ID hoặc upload JSON → chọn Prometheus datasource → Import.

---

## 7. Thêm target EC2 mới

### Bước 1: Thêm host vào inventory

Mở `inventory/production.yml`, thêm host mới vào `target_servers`:

```yaml
target_servers:
  hosts:
    # ... hosts hiện tại ...

    target-ec2-NEW-SERVER:                    # Tên hiển thị trên Grafana
      ansible_aws_ssm_instance_id: i-0xxxx   # Instance ID của EC2
      private_ip: "10.0.1.100"               # Private IP (Prometheus scrape qua IP này)
      # Bật/tắt exporter (xem mục 8 để chọn role phù hợp)
      install_node_exporter: true             # default: true
      install_cadvisor: true                  # default: true
      install_docker_exporter: false          # default: false
      install_phpfpm_exporter: false          # default: false
      install_magento_cron_monitor: false     # default: false
```

### Bước 2: Đảm bảo mạng thông

- Security Group của EC2 mới cho phép inbound port 9100, 8080, 9253 từ Monitoring EC2
- Nếu khác VPC: cần Transit Gateway + Route Table (xem mục 9)
- Test: từ Monitoring EC2, `curl http://<PRIVATE_IP>:9100/metrics`

### Bước 3: Deploy exporter lên target mới

```bash
source .env
ANSIBLE_CONFIG=./ansible.cfg ansible-playbook playbooks/setup-target.yml \
  -l target-ec2-NEW-SERVER \
  -i inventory/production.yml
```

### Bước 4: Update Prometheus config

```bash
source .env
ANSIBLE_CONFIG=./ansible.cfg ansible-playbook playbooks/setup-monitoring.yml \
  -i inventory/production.yml
```

Hoặc reload Prometheus không cần restart:

```bash
# SSH vào monitoring EC2
curl -X POST http://localhost:9090/-/reload
```

### Bước 5: Verify

- Prometheus Targets: http://18.138.145.51:9090/targets — target mới phải UP
- Grafana: chọn instance mới trong dropdown

---

## 8. Hướng dẫn chọn role phù hợp

### Bảng quyết định

| EC2 chạy gì? | Roles cần bật |
|---------------|---------------|
| Bất kỳ EC2 nào | `node_exporter` (default: true) |
| Có Docker containers (x86/amd64) | `cadvisor` (default: true) |
| Có Docker containers (ARM/Graviton hoặc Docker 29+ overlayfs) | `docker_exporter` + tắt `cadvisor` |
| Có PHP-FPM | `phpfpm_exporter` |
| Chạy Magento cron | `magento_cron_monitor` |
| Không có Docker | Tắt `cadvisor`: `install_cadvisor: false` |

### Chi tiết từng role

#### node_exporter (luôn bật)
- **Metrics**: CPU, RAM, Disk, Network, Systemd services
- **Port**: 9100
- **Không cần cấu hình thêm**

#### cadvisor
- **Metrics**: Docker container CPU, Memory, Network per container
- **Port**: 8080
- **Yêu cầu**: Docker đã cài
- **Lưu ý**: Không tương thích với Docker dùng `overlayfs` storage driver (Docker 29+, ARM Graviton). Kiểm tra: `docker info | grep "Storage Driver"`. Nếu là `overlayfs` → dùng `docker_exporter` thay thế.

#### docker_exporter (thay thế cAdvisor)
- **Metrics**: Container CPU %, Memory, Network (qua `docker stats`)
- **Port**: Không cần (đi qua node_exporter :9100 textfile collector)
- **Khi nào dùng**: EC2 ARM (Graviton) hoặc Docker dùng `overlayfs` storage driver
- **Cấu hình**:
```yaml
install_cadvisor: false
install_docker_exporter: true
```

#### phpfpm_exporter
- **Metrics**: PHP-FPM active processes, queue, slow requests, max children
- **Port**: 9253
- **Yêu cầu**: PHP-FPM đang chạy, status page enabled
- **Cấu hình**:
```yaml
install_phpfpm_exporter: true
```

#### magento_cron_monitor
- **Metrics**: Cron jobs by status, error details, error messages
- **Port**: Không cần (đi qua node_exporter :9100 textfile collector)
- **Yêu cầu**: mysql-client, kết nối được DB Magento
- **Cấu hình**: Set biến DB trong `.env`
```yaml
install_magento_cron_monitor: true
```

---

## 9. Monitor liên AWS Account

Khi EC2 Monitoring và EC2 Target nằm ở **khác AWS Account**, sử dụng **Transit Gateway** — thiết kế hub-and-spoke tập trung, cho phép nhiều AWS Account / VPC cắm vào dùng chung mà không cần tạo kết nối point-to-point giữa từng cặp VPC.

### 9.1 Kết nối mạng (Transit Gateway)

```
                        ┌──────────────────────────┐
                        │    Transit Gateway (TGW)  │
                        │    (Account A - Hub)      │
                        └─────┬──────┬──────┬──────┘
                              │      │      │
               TGW Attachment │      │      │ TGW Attachment
                              │      │      │
                    ┌─────────┘      │      └─────────┐
                    ▼                ▼                 ▼
  Account A (Monitoring)    Account B (Target)   Account C (Target)
  ┌──────────────────┐     ┌──────────────────┐  ┌──────────────────┐
  │ VPC: 10.0.0.0/16 │     │VPC: 20.10.0.0/16 │  │ VPC: 30.0.0.0/16 │
  │                  │     │                  │  │                  │
  │ EC2 Monitoring   │     │ EC2 Target       │  │ EC2 Target       │
  │ 10.0.1.50        │     │ 20.10.2.125      │  │ 30.0.1.50        │
  └──────────────────┘     └──────────────────┘  └──────────────────┘
```

> **Ưu điểm so với VPC Peering:**
> - Quản lý tập trung — thêm account/VPC mới chỉ cần attach vào TGW, không cần tạo N×(N-1)/2 peering connections
> - Hỗ trợ transitive routing — VPC A có thể đến VPC C thông qua TGW mà không cần kết nối trực tiếp
> - Dễ mở rộng khi có thêm nhiều AWS Account

**Bước 1**: Tạo Transit Gateway (trên Account A — hub)
- VPC Console → Transit Gateways → Create Transit Gateway
- Bật **Auto accept shared attachments** nếu muốn tự động accept
- Ghi lại TGW ID (ví dụ: `tgw-0abc123def456`)

**Bước 2**: Share TGW cho các Account khác (qua AWS RAM)
- AWS RAM Console → Create Resource Share
- Resource type: Transit Gateway → chọn TGW vừa tạo
- Thêm Account B, Account C, ... vào danh sách principals
- Account B, C: RAM Console → Accept resource share

**Bước 3**: Tạo Transit Gateway Attachment cho mỗi VPC
- Account A: VPC Console → Transit Gateway Attachments → Create
  - Transit Gateway: chọn TGW
  - VPC: chọn VPC monitoring (`10.0.0.0/16`)
  - Subnets: chọn subnet ở mỗi AZ
- Account B: Tương tự, attach VPC target (`20.10.0.0/16`) vào TGW
- Account C: Tương tự, attach VPC target (`30.0.0.0/16`) vào TGW

**Bước 4**: Cập nhật Route Tables
- Account A: Route Table → Add route:
  - `20.10.0.0/16` → Transit Gateway
  - `30.0.0.0/16` → Transit Gateway
- Account B: Route Table → Add route: `10.0.0.0/16` → Transit Gateway
- Account C: Route Table → Add route: `10.0.0.0/16` → Transit Gateway

**Bước 5**: Cập nhật Security Groups
- Account B (Target): Inbound rule cho port 9100, 8080, 9253 từ `10.0.0.0/16`
- Account C (Target): Tương tự

> **Thêm Account mới**: Chỉ cần Share TGW qua RAM → Tạo TGW Attachment → Cập nhật Route Table 2 bên → Cập nhật Security Group. Không cần thay đổi gì ở các Account đã có.

### 9.2 Ansible kết nối qua SSM

Ansible dùng **AWS SSM** để kết nối EC2 (không cần SSH). Khi target ở account khác:

**Cách 1: Cross-account SSM (khuyến nghị)**
- Account B tạo IAM Role cho phép Account A assume
- Cấu hình AWS Profile cho Account B:

```ini
# ~/.aws/config
[profile account-b]
role_arn = arn:aws:iam::<ACCOUNT_B_ID>:role/CrossAccountSSMRole
source_profile = default
region = ap-southeast-1
```

- Chạy Ansible với profile:

```bash
AWS_PROFILE=account-b ansible-playbook playbooks/setup-target.yml \
  -l target-ec2-NEW-SERVER \
  -i inventory/production.yml
```

**Cách 2: Dùng SSM bucket chung**
- Tạo S3 bucket cho phép cả 2 account access
- Set `ansible_aws_ssm_bucket_name` trong inventory

### 9.3 Inventory cho multi-account

```yaml
all:
  vars:
    ansible_connection: aws_ssm
    ansible_aws_ssm_region: ap-southeast-1

  children:
    monitoring_server:
      hosts:
        monitor-ec2:
          ansible_aws_ssm_instance_id: i-0xxxx  # Account A
          ansible_aws_ssm_bucket_name: "ansible-ssm-account-a"

    target_servers:
      hosts:
        # Target cùng account
        target-ec2-same-account:
          ansible_aws_ssm_instance_id: i-0yyyy
          ansible_aws_ssm_bucket_name: "ansible-ssm-account-a"
          private_ip: "10.0.1.100"

        # Target khác account
        target-ec2-other-account:
          ansible_aws_ssm_instance_id: i-0zzzz
          ansible_aws_ssm_bucket_name: "ansible-ssm-account-b"  # Bucket ở Account B
          private_ip: "20.10.2.100"
```

---

## 10. Vận hành & bảo trì

### Deploy chỉ 1 host

```bash
ansible-playbook playbooks/setup-target.yml \
  -l target-ec2-TS-Ecom-Gateway \
  -i inventory/production.yml
```

### Deploy chỉ 1 role

```bash
ansible-playbook playbooks/setup-target.yml \
  -l target-ec2-TS-M247-Cron \
  -i inventory/production.yml \
  --tags magento_cron_monitor
```

### Update Prometheus config (sau khi thêm/xóa target)

```bash
source .env
ansible-playbook playbooks/setup-monitoring.yml -i inventory/production.yml
```

Hoặc reload không restart:

```bash
# Trên monitoring EC2
curl -X POST http://localhost:9090/-/reload
```

### Gỡ exporter khỏi target

```bash
# Gỡ tất cả
ansible-playbook playbooks/cleanup-target.yml -i inventory/production.yml

# Gỡ exporter cụ thể
ansible-playbook playbooks/cleanup-target.yml --tags cadvisor
ansible-playbook playbooks/cleanup-target.yml --tags docker_exporter
ansible-playbook playbooks/cleanup-target.yml --tags phpfpm_exporter
ansible-playbook playbooks/cleanup-target.yml --tags magento_cron_monitor
```

### Gỡ monitoring stack

```bash
source .env
ansible-playbook playbooks/cleanup-monitoring.yml -i inventory/production.yml
```

### Metrics retention

Prometheus giữ metrics tối đa **7 ngày** hoặc **15GB** (cái nào chạm trước thì xóa).

---

## 11. Troubleshooting

### Prometheus target hiện DOWN

1. Kiểm tra exporter có chạy trên target: `curl http://localhost:9100/metrics`
2. Kiểm tra mạng từ monitoring → target: `curl http://<PRIVATE_IP>:9100/metrics`
3. Nếu timeout: kiểm tra Security Group, Route Table, Transit Gateway attachment
4. Kiểm tra UFW trên target: `sudo ufw status` → `sudo ufw allow 9100/tcp`

### cAdvisor không hiện container metrics

Docker dùng `overlayfs` storage driver (Docker 29+) không tương thích cAdvisor. Kiểm tra:

```bash
docker info | grep "Storage Driver"
```

Nếu là `overlayfs` → chuyển sang `docker_exporter`:

```yaml
# inventory
install_cadvisor: false
install_docker_exporter: true
```

### .env format lỗi trên WSL

```bash
sed -i 's/\r$//' .env
```

### Ansible không đọc ansible.cfg trên WSL

```bash
export ANSIBLE_CONFIG=./ansible.cfg
```

### Grafana không hiện data

1. Kiểm tra Data Source: Administration → Data Sources → Prometheus → Test
2. URL phải là `http://prometheus:9090` (Docker internal DNS)
3. Kiểm tra Prometheus Targets: http://18.138.145.51:9090/targets

### Alert email không gửi được

1. Kiểm tra SMTP credentials trong `.env`
2. Kiểm tra Alertmanager: http://18.138.145.51:9093
3. Test gửi: Alertmanager UI → Silence → tạo test alert

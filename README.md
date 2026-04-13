# Monitoring Setup - Ansible + Docker

Ansible playbooks để deploy monitoring stack (Prometheus + Grafana + Alertmanager) và exporters lên EC2 qua SSM.

## Kiến trúc tổng quan

```
EC2 Monitoring Server                    EC2 Target Servers
┌─────────────────────┐                 ┌──────────────────────┐
│  Prometheus (:9090)  │── scrape ──────│  node_exporter (:9100)│
│  Grafana    (:3000)  │                │  cAdvisor      (:8080)│
│  Alertmanager(:9093) │                │  phpfpm_export (:9253)│
└─────────────────────┘                 │  textfile collector   │
                                        └──────────────────────┘
```

## Cấu trúc project

```
├── .env.example                        # Template biến môi trường
├── .env                                # Biến môi trường (git ignored)
├── ansible.cfg
├── inventory/
│   ├── hosts.yml                       # Inventory chung
│   ├── production.yml                  # Inventory production
│   └── uat.yml                         # Inventory UAT
├── playbooks/
│   ├── setup-target.yml                # Cài exporters lên EC2 target
│   ├── setup-monitoring.yml            # Cài monitoring stack
│   ├── cleanup-target.yml              # Gỡ exporters
│   └── cleanup-monitoring.yml          # Gỡ monitoring stack
├── roles/
│   ├── node_exporter/                  # Host metrics (CPU, RAM, Disk) → port 9100
│   ├── cadvisor/                       # Docker container metrics → port 8080
│   ├── phpfpm_exporter/                # PHP-FPM metrics → port 9253
│   ├── docker_exporter/                # Docker metrics via textfile (thay thế cAdvisor cho ARM/overlayfs)
│   └── magento_cron_monitor/           # Magento cron job metrics via textfile
├── templates/
│   ├── prometheus.yml.j2               # Prometheus scrape config
│   └── alertmanager.yml.j2             # Alertmanager config
└── grafana/
    └── magento-cron-dashboard.json     # Grafana dashboard cho Magento Cron
```

## Roles & Exporters

### node_exporter
- **Port**: 9100
- **Mô tả**: Thu thập metrics hệ thống (CPU, RAM, Disk, Network, Systemd)
- **Cài đặt**: Binary + systemd service
- **Textfile collector**: `/var/lib/node_exporter/textfile_collector/` — các role khác ghi metrics vào đây

### cadvisor
- **Port**: 8080
- **Mô tả**: Thu thập Docker container metrics (CPU, Memory, Network per container)
- **Cài đặt**: Docker container
- **Lưu ý**: Không tương thích với Docker dùng `overlayfs` storage driver (Docker 29+ / containerd snapshotter). Xem [cadvisor#3860](https://github.com/google/cadvisor/issues/3860). Với các host bị ảnh hưởng, dùng `docker_exporter` thay thế.

### docker_exporter
- **Port**: Không cần (metrics đi qua node_exporter :9100)
- **Mô tả**: Thay thế cAdvisor cho các host không tương thích (ARM + Docker overlayfs driver). Dùng shell script + cron chạy `docker stats`, ghi ra textfile collector.
- **Cài đặt**: Script `/usr/local/bin/docker-container-metrics.sh` + cron mỗi phút
- **Metrics**: `container_cpu_usage_percent`, `container_memory_usage_bytes`, `container_last_seen`, `container_network_rx_bytes`, `container_network_tx_bytes`
- **Khi nào dùng**: Set `install_cadvisor: false` + `install_docker_exporter: true` trong inventory

### phpfpm_exporter
- **Port**: 9253
- **Mô tả**: Thu thập PHP-FPM metrics (active processes, queue, slow requests)
- **Cài đặt**: Binary + systemd service

### magento_cron_monitor
- **Port**: Không cần (metrics đi qua node_exporter :9100)
- **Mô tả**: Thu thập Magento cron job metrics từ DB `cron_schedule`
- **Cài đặt**: Script `/usr/local/bin/magento-cron-metrics.sh` + cron mỗi phút
- **Metrics**:
  - `magento_cron_jobs_total{status}` — Số jobs theo status (5 phút gần nhất)
  - `magento_cron_error_by_job{job_code}` — Số lỗi theo job (1 giờ gần nhất)
  - `magento_cron_error_detail{job_code,status,message,scheduled_at,executed_at}` — Chi tiết lỗi/missed mới nhất
  - `magento_cron_scrape_error` — Script có lỗi kết nối DB không
  - `magento_cron_last_scrape_timestamp` — Lần cuối script chạy
- **Yêu cầu**: mysql-client, kết nối DB (cấu hình qua env vars)

## Inventory

Mỗi host trong `target_servers` có thể bật/tắt từng exporter:

```yaml
target_servers:
  hosts:
    target-ec2-example:
      ansible_aws_ssm_instance_id: i-xxx
      private_ip: "10.0.1.100"
      # Flags bật/tắt (default trong ngoặc)
      install_node_exporter: true          # (default: true)
      install_cadvisor: true               # (default: true)
      install_docker_exporter: false       # (default: false)
      install_phpfpm_exporter: false       # (default: false)
      install_magento_cron_monitor: false  # (default: false)
```

### Hosts hiện tại (Production)

| Host | IP | Exporters |
|------|----|-----------|
| target-ec2-ts-oms | 20.10.11.56 | node_exporter, cadvisor |
| target-ec2-TS-M247-Prod | 20.10.2.121 | node_exporter, cadvisor, phpfpm |
| target-ec2-TS-M247-Prod-2 | 20.10.2.243 | node_exporter, cadvisor, phpfpm |
| target-ec2-TS-Ecom-Gateway | 20.10.2.125 | node_exporter, docker_exporter |
| target-ec2-TS-M247-Cron | 20.10.2.35 | node_exporter, magento_cron_monitor |

## Prerequisites

Chạy trên Ubuntu WSL:

```bash
cd "/mnt/c/your_local_path"

# Ansible + dependencies
sudo apt install python3-pip python3-boto3 -y
pip install ansible

# SSM connection plugin
ansible-galaxy collection install community.aws

# AWS Session Manager Plugin
curl "https://s3.amazonaws.com/session-manager-downloads/plugin/latest/ubuntu_64bit/session-manager-plugin.deb" -o /tmp/session-manager-plugin.deb
sudo dpkg -i /tmp/session-manager-plugin.deb
rm /tmp/session-manager-plugin.deb
```

Verify:

```bash
ansible --version
python3 -c "import boto3; print(boto3.__version__)"
ansible-galaxy collection list | grep community.aws
session-manager-plugin --version
aws sts get-caller-identity
```

## Cấu hình

1. Copy `.env.example` → `.env` và điền thông tin:

```bash
cp .env.example .env
```

2. Sửa inventory file (`inventory/production.yml` hoặc `inventory/uat.yml`):
   - `ansible_aws_ssm_instance_id`: Instance ID của các EC2
   - `private_ip`: Private IP để Prometheus scrape
   - `ansible_aws_ssm_bucket_name`: S3 bucket cho SSM file transfer
   - Flags bật/tắt exporter cho từng host

## Cách chạy

> **Lưu ý WSL**: Thư mục `/mnt/c/...` là world-writable nên Ansible bỏ qua `ansible.cfg`.
> Luôn thêm `ANSIBLE_CONFIG=./ansible.cfg` hoặc chạy `export ANSIBLE_CONFIG=./ansible.cfg` trước.

### Deploy exporters lên target

```bash
# Tất cả targets
ANSIBLE_CONFIG=./ansible.cfg ansible-playbook playbooks/setup-target.yml -i inventory/production.yml

# Chỉ 1 host
ansible-playbook playbooks/setup-target.yml -l target-ec2-TS-Ecom-Gateway -i inventory/production.yml

# Chỉ 1 role
ansible-playbook playbooks/setup-target.yml -l target-ec2-TS-M247-Cron -i inventory/production.yml --tags magento_cron_monitor
```

### Deploy monitoring stack

```bash
source .env && ANSIBLE_CONFIG=./ansible.cfg ansible-playbook playbooks/setup-monitoring.yml -i inventory/production.yml
```

### Cleanup

```bash
# Gỡ tất cả exporters
ansible-playbook playbooks/cleanup-target.yml -i inventory/production.yml

# Gỡ exporter cụ thể
ansible-playbook playbooks/cleanup-target.yml --tags cadvisor
ansible-playbook playbooks/cleanup-target.yml --tags docker_exporter
ansible-playbook playbooks/cleanup-target.yml --tags phpfpm_exporter
ansible-playbook playbooks/cleanup-target.yml --tags magento_cron_monitor

# Gỡ monitoring stack
source .env && ansible-playbook playbooks/cleanup-monitoring.yml -i inventory/production.yml
```

## Grafana Dashboards

### Import sẵn từ Grafana.com
| ID | Dashboard | Dùng cho |
|----|-----------|----------|
| 1860 | Node Exporter Full | Host metrics |
| 14282 | cAdvisor Docker | Docker containers (hosts dùng cAdvisor) |
| 4912 | PHP-FPM | PHP-FPM metrics |

### Custom dashboards
| File | Dashboard | Dùng cho |
|------|-----------|----------|
| `grafana/magento-cron-dashboard.json` | Magento Cron Monitor | Cron job monitoring |

Import: Grafana → Dashboards → New → Import → Upload JSON file

### Grafana queries cho host dùng docker_exporter

Các host dùng `docker_exporter` thay vì cAdvisor cần query khác:

**Memory Usage (MB):**
```promql
container_memory_usage_bytes{name!="", name!="cadvisor", name!="docker_exporter", instance_name=~"$instance_name"} / 1024 / 1024
```

**CPU Usage (%):**
```promql
topk(10, sum by (name, instance_name) (rate(container_cpu_usage_seconds_total{name!="", name!="cadvisor", instance_name=~"$instance_name"}[5m])) * 100)
or
topk(10, container_cpu_usage_percent{name!="", name!="docker_exporter", instance_name=~"$instance_name"})
```

**Docker Container Status:**
```promql
max by(name) (clamp(delta(container_last_seen{name!="", name!="cadvisor", name!="docker_exporter", instance_name=~"$instance_name"}[2m]), 0, 1)
OR
clamp_max(container_last_seen{name!="", name!="cadvisor", name!="docker_exporter", instance_name=~"$instance_name"} * 0, 0))
```

## Alert Rules

Alerts được cấu hình trong `playbooks/setup-monitoring.yml`:

| Alert | Điều kiện | Severity |
|-------|-----------|----------|
| TargetDown | Exporter không scrape được (trừ cAdvisor, phpfpm) | critical |
| CAdvisorDown | cAdvisor không scrape được | warning |
| HighCPU | CPU > 80% trong 5 phút | warning |
| HighMemory | Memory > 85% trong 5 phút | warning |
| DiskSpaceLow | Disk > 85% trong 5 phút | critical |
| ContainerNotRunning | Container biến mất > 2 phút | critical |
| ContainerHighCPU | Container CPU > 80% (cAdvisor hoặc textfile) | warning |
| MagentoCronHighErrors | Cron error jobs > 5 trong 5 phút | warning |
| MagentoCronNotRunning | Script metrics không chạy > 5 phút | critical |

## Security Groups

| EC2 | Port | Source | Service |
|-----|------|--------|---------|
| Target | 9100 | Monitoring SG | node_exporter |
| Target | 8080 | Monitoring SG | cAdvisor |
| Target | 9253 | Monitoring SG | phpfpm_exporter |
| Monitoring | 9090 | Your IP / VPN | Prometheus |
| Monitoring | 3000 | ALB SG / Your IP | Grafana |
| Monitoring | 9093 | Your IP / VPN | Alertmanager |

## Metrics Retention

Prometheus giữ metrics tối đa 7 ngày hoặc 15GB (cái nào chạm trước thì xóa):

```yaml
- '--storage.tsdb.retention.time=7d'
- '--storage.tsdb.retention.size=15GB'
```

## Troubleshooting

### .env format lỗi trên WSL
```bash
sed -i 's/\r$//' .env
```

### cAdvisor không hiện container metrics
Nếu Docker dùng `overlayfs` storage driver (check: `docker info | grep "Storage Driver"`), cAdvisor không tương thích. Chuyển sang dùng `docker_exporter`:
```yaml
# inventory
install_cadvisor: false
install_docker_exporter: true
```

### Prometheus reload config không cần restart
```bash
curl -X POST http://localhost:9090/-/reload
```

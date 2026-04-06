# Monitoring Setup - Ansible + Docker

## Cấu trúc

```
monitoring-setup/
├── ansible.cfg
├── inventory/
│   └── hosts.yml              # Inventory với SSM connection
├── playbooks/
│   ├── setup-target.yml       # Cài exporters lên EC2 target
│   └── setup-monitoring.yml   # Cài Prometheus+Grafana+Alertmanager (Docker)
└── roles/
    ├── node_exporter/         # Host metrics (CPU, RAM, Disk)
    ├── cadvisor/              # Docker container metrics
    └── phpfpm_exporter/       # PHP-FPM metrics
```

## Prerequisites

```bash
# Cài Ansible + SSM plugin
pip install ansible boto3

# Cài SSM connection plugin
ansible-galaxy collection install community.aws

# Cài AWS Session Manager Plugin
# https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html

# Đảm bảo AWS credentials đã config
aws sts get-caller-identity
```

## Cách chạy

### Bước 1: Cài exporters lên EC2 target (i-03a7cd36465405052)

```bash
cd monitoring-setup

# Chạy playbook (đổi phpfpm vars nếu cần)
ansible-playbook playbooks/setup-target.yml

# Nếu php-fpm listen trên unix socket:
ansible-playbook playbooks/setup-target.yml \
  -e 'phpfpm_scrape_uri="unix:///var/run/php/php-fpm.sock;/status"' \
  -e 'phpfpm_pool_config=/etc/php/8.3/fpm/pool.d/www.conf'
```

### Bước 2: Cài monitoring stack lên EC2 monitoring (i-07f529f2770a5bfa5)

```bash
# Thay PRIVATE_IP bằng private IP của EC2 target
ansible-playbook playbooks/setup-monitoring.yml \
  -e 'target_ip=10.0.x.x'
```

### Bước 3: Truy cập Grafana

1. Mở `http://<MONITORING_EC2_PUBLIC_IP>:3000`
2. Login: `admin` / `admin`
3. Add Data Source → Prometheus → URL: `http://prometheus:9090`
4. Import Dashboard:
   - ID `1860` → Node Exporter Full
   - ID `14282` → cAdvisor Docker
   - ID `4912` → PHP-FPM

## Security Groups cần mở

### EC2 Target (i-03a7cd36465405052)
| Port | Source | Service |
|------|--------|---------|
| 9100 | Monitoring EC2 SG | node_exporter |
| 8080 | Monitoring EC2 SG | cAdvisor |
| 9253 | Monitoring EC2 SG | php-fpm_exporter |

### EC2 Monitoring (i-07f529f2770a5bfa5)
| Port | Source | Service |
|------|--------|---------|
| 9090 | Your IP | Prometheus |
| 3000 | Your IP | Grafana |
| 9093 | Your IP | Alertmanager |

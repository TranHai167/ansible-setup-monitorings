# Monitoring Setup - Ansible + Docker

Ansible playbooks để deploy monitoring stack (Prometheus + Grafana + Alertmanager) và exporters lên EC2 qua SSM.

## Cấu trúc

```
├── .env.example               # Template biến môi trường
├── .env                       # Biến môi trường (git ignored)
├── ansible.cfg
├── inventory/
│   └── hosts.yml              # Inventory - SSM connection
├── playbooks/
│   ├── setup-target.yml       # Cài exporters lên EC2 target (Ubuntu)
│   └── setup-monitoring.yml   # Cài monitoring stack lên EC2 (Amazon Linux 2023)
└── roles/
    ├── node_exporter/         # CPU, RAM, Disk metrics (:9100)
    ├── cadvisor/              # Docker container metrics (:8080)
    └── phpfpm_exporter/       # PHP-FPM metrics (:9253)
```

## Prerequisites

Chạy trên Ubuntu WSL:

Lưu ý: move to correct directory:

```bash
cd "/mnt/c/your_local_path"
```

```bash
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

```env
SES_SMTP_USERNAME=           # AWS SES SMTP credentials
SES_SMTP_PASSWORD=
FROM_EMAIL=cloudtrail@sts.vn
TO_EMAIL=awscloudtraillogs@sts.vn
TARGET_IP=10.0.3.130         # Private IP của EC2 target
```

2. Sửa `inventory/hosts.yml`:
   - `ansible_aws_ssm_instance_id`: Instance ID của các EC2
   - `ansible_aws_ssm_bucket_name`: Tên S3 bucket cho SSM file transfer
   - `ansible_aws_ssm_region`: Region

## Cách chạy

> **Lưu ý WSL**: Thư mục `/mnt/c/...` là world-writable nên Ansible bỏ qua `ansible.cfg`.
> Luôn thêm `ANSIBLE_CONFIG=./ansible.cfg` hoặc chạy `export ANSIBLE_CONFIG=./ansible.cfg` trước.

### Bước 1: Cài exporters lên EC2 target

```bash
ANSIBLE_CONFIG=./ansible.cfg ansible-playbook playbooks/setup-target.yml
```

### Bước 2: Cài monitoring stack

```bash
source .env && ANSIBLE_CONFIG=./ansible.cfg ansible-playbook playbooks/setup-monitoring.yml
```

### Bước 3: Verify

Trên EC2 target:

```bash
sudo ss -tlnp | grep -E ':(9100|8080|9253)'
curl -s http://localhost:9253/metrics | grep phpfpm_up
```

Trên EC2 monitoring:

```bash
docker ps
curl -s http://localhost:9090/-/healthy
curl -s http://localhost:3000/api/health
```

### Bước 4: Truy cập Grafana

1. Mở `http://<MONITORING_EC2_IP>:3000` — login `admin` / `admin`
2. Add Data Source → Prometheus → URL: `http://prometheus:9090`
3. Import Dashboard:
   - `1860` — Node Exporter Full
   - `14282` — cAdvisor Docker
   - `4912` — PHP-FPM

## Security Groups

| EC2 | Port | Source | Service |
|-----|------|--------|---------|
| Target | 9100 | Monitoring SG | node_exporter |
| Target | 8080 | Monitoring SG | cAdvisor |
| Target | 9253 | Monitoring SG | php-fpm_exporter |
| Monitoring | 9090 | Your IP / VPN | Prometheus |
| Monitoring | 3000 | ALB SG / Your IP | Grafana |
| Monitoring | 9093 | Your IP / VPN | Alertmanager |

## Metrics Retention

Prometheus giữ metrics tối đa 7 ngày hoặc 15GB (cái nào chạm trước thì xóa). Cấu hình trong `setup-monitoring.yml`:

```yaml
- '--storage.tsdb.retention.time=7d'
- '--storage.tsdb.retention.size=15GB'
```

If format in env is incorrect, use:

``` bash
sed -i 's/\r$//' .env
```

If format can not process, fix this content:

```bash

cat > /opt/monitoring/alertmanager/alertmanager.yml << 'EOF'
route:
  receiver: email
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  routes:
    - match:
        severity: critical
      repeat_interval: 1h
receivers:
  - name: email
    email_configs:
      - to: "haitn2@sts.vn, cuongph@sts.vn"
        from: "no_reply_test@sts.vn"
        smarthost: smtp.office365.com:587
        auth_username: "no_reply_test@sts.vn"
        auth_password: "Xuw76289"
        auth_identity: "no_reply_test@sts.vn"
        require_tls: true
        send_resolved: true
        headers:
          Subject: '[{{ .Status | toUpper }}] {{ .GroupLabels.alertname }}'
EOF

```


Gỡ exporter trên target EC2:

```bash
# Gỡ tất cả
source .env && AWS_PROFILE=magento-conf ANSIBLE_CONFIG=./ansible.cfg ansible-playbook playbooks/cleanup-target.yml

# Chỉ gỡ 1 exporter cụ thể
ansible-playbook playbooks/cleanup-target.yml --tags node_exporter
ansible-playbook playbooks/cleanup-target.yml --tags cadvisor
ansible-playbook playbooks/cleanup-target.yml --tags phpfpm_exporter
```

Gỡ monitoring stack (Prometheus, Grafana, Alertmanager):
```
source .env && AWS_PROFILE=magento-conf ANSIBLE_CONFIG=./ansible.cfg ansible-playbook playbooks/cleanup-monitoring.yml
```

Allow ufw rule for node_exporter port:
```
sudo ufw allow 9100/tcp
```

aws ssm start-session --target i-0ae93a49a72bc8557  --document-name AWS-StartPortForwardingSessionToRemoteHost --parameters "host=['ecom-proxydb.proxy-cl2l6b5wfjrb.ap-southeast-1.rds.amazonaws.com'],portNumber=['3306'],localPortNumber=['3306']"

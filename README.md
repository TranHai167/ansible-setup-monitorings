# Monitoring Setup - Ansible + Docker

## Cấu trúc

```
monitoring-setup/
├── ansible.cfg
├── inventory/
│   └── hosts.yml              # Inventory với SSM connection
├── playbooks/
│   ├── setup-target.yml       # Cài exporters lên EC2 target (Ubuntu)
│   └── setup-monitoring.yml   # Cài Prometheus+Grafana+Alertmanager (Amazon Linux 2023)
└── roles/
    ├── node_exporter/         # Host metrics (CPU, RAM, Disk)
    ├── cadvisor/              # Docker container metrics
    └── phpfpm_exporter/       # PHP-FPM metrics
```

## Prerequisites

```bash
# Cài Ansible + dependencies
sudo apt install python3-pip python3-boto3 -y
pip install ansible
# Hoặc: sudo apt install ansible

# Cài SSM connection plugin
ansible-galaxy collection install community.aws

# Cài AWS Session Manager Plugin
curl "https://s3.amazonaws.com/session-manager-downloads/plugin/latest/ubuntu_64bit/session-manager-plugin.deb" -o /tmp/session-manager-plugin.deb
sudo dpkg -i /tmp/session-manager-plugin.deb
rm /tmp/session-manager-plugin.deb
```

### Kiểm tra điều kiện tiên quyết

```bash
python3 --version
ansible --version
python3 -c "import boto3; print(boto3.__version__)"
ansible-galaxy collection list | grep community.aws
session-manager-plugin --version
aws sts get-caller-identity
```

## Lưu ý quan trọng

- Thư mục project nằm trên `/mnt/c/...` (WSL mount) là world-writable, Ansible sẽ bỏ qua `ansible.cfg`.
  Luôn chạy với prefix: `ANSIBLE_CONFIG=./ansible.cfg` hoặc `export ANSIBLE_CONFIG=./ansible.cfg` trước.
- EC2 target: **Ubuntu** (dùng apt)
- EC2 monitoring: **Amazon Linux 2023** (dùng dnf)
- Cần cấu hình `ansible_aws_ssm_bucket_name` trong `inventory/hosts.yml` với tên S3 bucket thực tế.

## Cách chạy

### Bước 1: Cài exporters lên EC2 target

```bash
ANSIBLE_CONFIG=./ansible.cfg ansible-playbook playbooks/setup-target.yml
```

Override php-fpm config nếu cần:

```bash
ANSIBLE_CONFIG=./ansible.cfg ansible-playbook playbooks/setup-target.yml \
  -e 'phpfpm_scrape_uri="unix:///var/run/php/php-fpm.sock;/status"' \
  -e 'phpfpm_pool_config=/etc/php/8.3/fpm/pool.d/www.conf'
```

### Bước 2: Cài monitoring stack lên EC2 monitoring

```bash
# Thay 10.0.3.130 bằng private IP thực của EC2 target
ANSIBLE_CONFIG=./ansible.cfg ansible-playbook playbooks/setup-monitoring.yml \
  -e 'target_ip=10.0.3.130'
```

Nếu dùng Alertmanager gửi email qua SES:

```bash
export SES_SMTP_USERNAME="your_ses_smtp_user"
export SES_SMTP_PASSWORD="your_ses_smtp_pass"
```

### Bước 3: Verify trên EC2 target

```bash
sudo ss -tlnp | grep -E ':(9100|8080|9253)'
curl -s http://localhost:9100/metrics | head -5
curl -s http://localhost:8080/metrics | head -5
curl -s http://localhost:9253/metrics | grep phpfpm_up
```

### Bước 4: Truy cập Grafana

1. Mở `http://<MONITORING_EC2_IP>:3000`
2. Login: `admin` / `admin`
3. Add Data Source → Prometheus → URL: `http://prometheus:9090`
4. Import Dashboard:
   - ID `1860` → Node Exporter Full
   - ID `14282` → cAdvisor Docker
   - ID `4912` → PHP-FPM

## Security Groups cần mở

### EC2 Target
| Port | Source | Service |
|------|--------|---------|
| 9100 | Monitoring EC2 SG | node_exporter |
| 8080 | Monitoring EC2 SG | cAdvisor |
| 9253 | Monitoring EC2 SG | php-fpm_exporter |

### EC2 Monitoring
| Port | Source | Service |
|------|--------|---------|
| 9090 | Your IP | Prometheus |
| 3000 | Your IP | Grafana |
| 9093 | Your IP | Alertmanager |

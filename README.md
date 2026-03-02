# دليل شامل: أدوات DevOps - الجزء الثاني

> **التاريخ:** مارس 2026  
> **الغرض:** استكمال أدوات DevOps (المراقبة، الأمان، القواعد، والحلول Offline)  
> **ملاحظة:** ⚠️ = يتطلب اتصال إنترنت | 📦 = يمكن تشغيله Offline

---

## 📋 جدول المحتويات

1. [المراقبة والسجلات](#المراقبة-والسجلات)
2. [الأمان وإدارة الأسرار](#الأمان-وإدارة-الأسرار)
3. [قواعد البيانات](#قواعد-البيانات)
4. [أدوات البناء والتطوير](#أدوات-البناء-والتطوير)
5. [أدوات الاختبار](#أدوات-الاختبار)
6. [Service Mesh و API Gateway](#service-mesh-و-api-gateway)
7. [إدارة المشاريع والتعاون](#إدارة-المشاريع-والتعاون)
8. [منصات السحابة](#منصات-السحابة)
9. [استراتيجية Offline Toolkit الشاملة](#استراتيجية-offline-toolkit-الشاملة)
10. [خارطة طريق التنفيذ](#خارطة-طريق-التنفيذ)

---

## 1. المراقبة والسجلات

### 1.1 Prometheus - نظام المراقبة

| المكون | الوصف | Offline |
|--------|-------|---------|
| **Prometheus** 📦 | نظام مراقبة وتنبيهات | ✅ |
| **Alertmanager** 📦 | إدارة التنبيهات | ✅ |
| **Pushgateway** 📦 | لـ batch jobs | ✅ |
| **Node Exporter** 📦 | metrics للسيرفرات | ✅ |
| **Blackbox Exporter** 📦 | فحص endpoints | ✅ |

#### تثبيت Prometheus Stack Offline

```bash
# على جهاز متصل - تحميل binaries
wget https://github.com/prometheus/prometheus/releases/download/v2.50.1/prometheus-2.50.1.linux-amd64.tar.gz
wget https://github.com/prometheus/alertmanager/releases/download/v0.27.0/alertmanager-0.27.0.linux-amd64.tar.gz
wget https://github.com/prometheus/node_exporter/releases/download/v1.7.0/node_exporter-1.7.0.linux-amd64.tar.gz
wget https://github.com/prometheus/pushgateway/releases/download/v1.7.0/pushgateway-1.7.0.linux-amd64.tar.gz
wget https://github.com/prometheus/blackbox_exporter/releases/download/v0.24.0/blackbox_exporter-0.24.0.linux-amd64.tar.gz

# على جهاز Offline
# 1. Prometheus
tar -xzf prometheus-2.50.1.linux-amd64.tar.gz
cd prometheus-2.50.1.linux-amd64
sudo cp prometheus promtool /usr/local/bin/
sudo mkdir -p /etc/prometheus /var/lib/prometheus
sudo cp prometheus.yml /etc/prometheus/

# إنشاء systemd service
sudo tee /etc/systemd/system/prometheus.service > /dev/null <<EOF
[Unit]
Description=Prometheus
After=network.target

[Service]
Type=simple
User=prometheus
ExecStart=/usr/local/bin/prometheus \\
  --config.file=/etc/prometheus/prometheus.yml \\
  --storage.tsdb.path=/var/lib/prometheus \\
  --web.console.templates=/etc/prometheus/consoles \\
  --web.console.libraries=/etc/prometheus/console_libraries
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

sudo useradd -rs /bin/false prometheus
sudo chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus
sudo systemctl enable --now prometheus

# 2. Node Exporter (على كل node)
tar -xzf node_exporter-1.7.0.linux-amd64.tar.gz
sudo cp node_exporter-1.7.0.linux-amd64/node_exporter /usr/local/bin/

sudo tee /etc/systemd/system/node_exporter.service > /dev/null <<EOF
[Unit]
Description=Node Exporter
After=network.target

[Service]
Type=simple
User=node_exporter
ExecStart=/usr/local/bin/node_exporter
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

sudo useradd -rs /bin/false node_exporter
sudo systemctl enable --now node_exporter

# 3. Alertmanager
tar -xzf alertmanager-0.27.0.linux-amd64.tar.gz
cd alertmanager-0.27.0.linux-amd64
sudo cp alertmanager amtool /usr/local/bin/
sudo mkdir -p /etc/alertmanager /var/lib/alertmanager

# إعداد التهيئة
sudo tee /etc/alertmanager/alertmanager.yml > /dev/null <<EOF
global:
  resolve_timeout: 5m

route:
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h
  receiver: 'team-email'

receivers:
- name: 'team-email'
  email_configs:
  - to: 'team@example.com'
    from: 'alertmanager@example.com'
    smarthost: localhost:25
    require_tls: false
EOF

sudo systemctl enable --now alertmanager
```

### 1.2 Grafana - لوحات المراقبة

| المكون | الوصف | Offline |
|--------|-------|---------|
| **Grafana** 📦 | لوحات تحكم | ✅ |
| **Grafana Loki** 📦 | سجلات مجمعة | ✅ |
| **Grafana Tempo** 📦 | distributed tracing | ✅ |

#### تثبيت Grafana Offline

```bash
# على جهاز متصل
# Ubuntu
wget https://dl.grafana.com/oss/release/grafana_10.3.3_amd64.deb

# Rocky Linux
wget https://dl.grafana.com/oss/release/grafana-10.3.3-1.x86_64.rpm

# تحميل plugins يدوياً
wget https://grafana.com/api/plugins/grafana-piechart-panel/versions/1.6.2/download -O grafana-piechart-panel.zip

# على جهاز Offline
# Ubuntu
sudo dpkg -i grafana_10.3.3_amd64.deb

# Rocky Linux
sudo rpm -ivh grafana-10.3.3-1.x86_64.rpm

# تثبيت plugins
sudo grafana-cli --pluginsDir /var/lib/grafana/plugins plugins install grafana-piechart-panel

# أو يدوياً
sudo mkdir -p /var/lib/grafana/plugins
sudo unzip grafana-piechart-panel.zip -d /var/lib/grafana/plugins/
sudo chown -R grafana:grafana /var/lib/grafana/plugins

sudo systemctl enable --now grafana-server

# الوصول: http://localhost:3000
# Default: admin/admin
```

### 1.3 ELK Stack - السجلات المركزية

| المكون | الوصف | Offline |
|--------|-------|---------|
| **Elasticsearch** 📦 | محرك البحث | ✅ |
| **Logstash** 📦 | معالج السجلات | ✅ |
| **Kibana** 📦 | واجهة التحليل | ✅ |
| **Filebeat** 📦 | shipper خفيف | ✅ |
| **Metricbeat** 📦 | metrics shipper | ✅ |

#### تثبيت ELK Stack Offline

```bash
# على جهاز متصل
# تحميل Elasticsearch
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-8.12.2-linux-x86_64.tar.gz

# تحميل Kibana
wget https://artifacts.elastic.co/downloads/kibana/kibana-8.12.2-linux-x86_64.tar.gz

# تحميل Logstash
wget https://artifacts.elastic.co/downloads/logstash/logstash-8.12.2-linux-x86_64.tar.gz

# تحميل Filebeat
wget https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-8.12.2-linux-x86_64.tar.gz

# على جهاز Offline
# 1. Elasticsearch
tar -xzf elasticsearch-8.12.2-linux-x86_64.tar.gz
sudo mv elasticsearch-8.12.2 /opt/elasticsearch

# إعداد
sudo useradd -r elasticsearch
sudo chown -R elasticsearch:elasticsearch /opt/elasticsearch

# إنشاء service
sudo tee /etc/systemd/system/elasticsearch.service > /dev/null <<EOF
[Unit]
Description=Elasticsearch
After=network.target

[Service]
Type=simple
User=elasticsearch
ExecStart=/opt/elasticsearch/bin/elasticsearch
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF

# تعديل الإعدادات للبيئة Offline
nano /opt/elasticsearch/config/elasticsearch.yml
# network.host: 0.0.0.0
# discovery.type: single-node
# xpack.security.enabled: false

sudo systemctl enable --now elasticsearch

# 2. Kibana
tar -xzf kibana-8.12.2-linux-x86_64.tar.gz
sudo mv kibana-8.12.2 /opt/kibana
sudo useradd -r kibana
sudo chown -R kibana:kibana /opt/kibana

# تعديل الإعدادات
nano /opt/kibana/config/kibana.yml
# server.host: "0.0.0.0"
# elasticsearch.hosts: ["http://localhost:9200"]

sudo tee /etc/systemd/system/kibana.service > /dev/null <<EOF
[Unit]
Description=Kibana
After=network.target elasticsearch.service

[Service]
Type=simple
User=kibana
ExecStart=/opt/kibana/bin/kibana
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl enable --now kibana

# 3. Filebeat (على كل node)
tar -xzf filebeat-8.12.2-linux-x86_64.tar.gz
sudo mv filebeat-8.12.2-linux-x86_64 /opt/filebeat

# إعداد
nano /opt/filebeat/filebeat.yml
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /var/log/*.log

output.elasticsearch:
  hosts: ["http://elasticsearch-server:9200"]

sudo /opt/filebeat/filebeat -e -c /opt/filebeat/filebeat.yml
```

### 1.4 بدائل المراقبة

| الأداة | الوصف | Offline |
|-------|-------|---------|
| **Zabbix** 📦 | مراقبة شاملة | ✅ |
| **Nagios** 📦 | مراقبة كلاسيكية | ✅ |
| **Netdata** 📦 | real-time monitoring | ✅ |
| **VictoriaMetrics** 📦 | بديل Prometheus | ✅ |
| **Graylog** 📦 | إدارة سجلات | ✅ |
| **Fluentd** 📦 | جامع سجلات | ✅ |

---

## 2. الأمان وإدارة الأسرار

### 2.1 HashiCorp Vault - إدارة الأسرار

| المكون | الوصف | Offline |
|--------|-------|---------|
| **Vault** 📦 | إدارة أسرار | ✅ |
| **Vault Agent** 📦 | client agent | ✅ |

#### تثبيت Vault Offline

```bash
# على جهاز متصل
wget https://releases.hashicorp.com/vault/1.15.5/vault_1.15.5_linux_amd64.zip
unzip vault_1.15.5_linux_amd64.zip

# على جهاز Offline
sudo mv vault /usr/local/bin/
sudo chmod +x /usr/local/bin/vault

# إعداد Vault Server
sudo useradd -r vault
sudo mkdir -p /etc/vault /var/lib/vault

# إنشاء ملف التهيئة
sudo tee /etc/vault/config.hcl > /dev/null <<EOF
storage "file" {
  path = "/var/lib/vault/data"
}

listener "tcp" {
  address     = "0.0.0.0:8200"
  tls_disable = 1
}

api_addr = "http://127.0.0.1:8200"
cluster_addr = "https://127.0.0.1:8201"
ui = true
EOF

sudo chown -R vault:vault /etc/vault /var/lib/vault

# إنشاء systemd service
sudo tee /etc/systemd/system/vault.service > /dev/null <<EOF
[Unit]
Description=HashiCorp Vault
After=network.target

[Service]
Type=notify
User=vault
ExecStart=/usr/local/bin/vault server -config=/etc/vault/config.hcl
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl enable --now vault

# تهيئة Vault
export VAULT_ADDR='http://127.0.0.1:8200'
vault operator init
# حفظ unseal keys و root token

# فك التشفير
vault operator unseal <key1>
vault operator unseal <key2>
vault operator unseal <key3>

# تسجيل الدخول
vault login <root_token>

# إنشاء secret
vault kv put secret/myapp/config username="admin" password="secret123"

# قراءة secret
vault kv get secret/myapp/config
```

### 2.2 أدوات أمان أخرى

| الأداة | الوصف | Offline |
|-------|-------|---------|
| **Sealed Secrets** 📦 | تشفير K8s secrets | ✅ |
| **SOPS** 📦 | تشفير ملفات | ✅ |
| **age** 📦 | تشفير بسيط | ✅ |
| **Trivy** 📦 | فحص ثغرات | 🔄 DB محلي |
| **Clair** 📦 | فحص container images | 🔄 DB محلي |
| **Anchore** 📦 | أمان الحاويات | 🔄 |
| **Falco** 📦 | runtime security | ✅ |
| **OWASP ZAP** 📦 | security testing | ✅ |
| **SonarQube** 📦 | code quality | ✅ |

#### Trivy Offline - فحص الثغرات

```bash
# على جهاز متصل
wget https://github.com/aquasecurity/trivy/releases/download/v0.49.1/trivy_0.49.1_Linux-64bit.tar.gz
tar -xzf trivy_0.49.1_Linux-64bit.tar.gz

# تحميل vulnerability database
trivy image --download-db-only

# نسخ DB
cp -r ~/.cache/trivy ~/trivy-db-offline

# على جهاز Offline
sudo mv trivy /usr/local/bin/
cp -r ~/trivy-db-offline ~/.cache/trivy

# استخدام
trivy image --skip-update nginx:latest
```

### 2.3 Certificate Management

| الأداة | الوصف | Offline |
|-------|-------|---------|
| **cert-manager** 📦 | شهادات K8s | 🔄 |
| **Let's Encrypt** ⚠️ | شهادات مجانية | ❌ |
| **step-ca** 📦 | CA داخلي | ✅ |
| **CFSSL** 📦 | PKI toolkit | ✅ |

---

## 3. قواعد البيانات

### 3.1 قواعد البيانات العلائقية

| القاعدة | Container Image | Offline |
|---------|----------------|---------|
| **PostgreSQL** 📦 | postgres:16 | ✅ |
| **MySQL** 📦 | mysql:8 | ✅ |
| **MariaDB** 📦 | mariadb:11 | ✅ |
| **Oracle DB** ⚠️ | - | محدود |

#### تشغيل PostgreSQL Offline

```bash
# على جهاز متصل
docker pull postgres:16
docker save postgres:16 -o postgres16.tar

# على جهاز Offline
docker load -i postgres16.tar

# تشغيل
docker run -d \
  --name postgres \
  -e POSTGRES_PASSWORD=mypassword \
  -v /data/postgres:/var/lib/postgresql/data \
  -p 5432:5432 \
  postgres:16

# أو باستخدام Docker Compose
cat > docker-compose.yml <<EOF
version: '3.8'
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: mypassword
      POSTGRES_USER: admin
      POSTGRES_DB: myapp
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    restart: always

volumes:
  postgres_data:
EOF

docker-compose up -d
```

### 3.2 قواعد البيانات NoSQL

| القاعدة | Container Image | Offline |
|---------|----------------|---------|
| **MongoDB** 📦 | mongo:7 | ✅ |
| **Redis** 📦 | redis:7 | ✅ |
| **Cassandra** 📦 | cassandra:5 | ✅ |
| **CouchDB** 📦 | couchdb:3 | ✅ |
| **InfluxDB** 📦 | influxdb:2 | ✅ |

### 3.3 أدوات إدارة قواعد البيانات

| الأداة | الوصف | Offline |
|-------|-------|---------|
| **pgAdmin** 📦 | إدارة PostgreSQL | ✅ |
| **phpMyAdmin** 📦 | إدارة MySQL | ✅ |
| **DBeaver** 📦 | multi-DB client | ✅ |
| **Adminer** 📦 | إدارة DB خفيفة | ✅ |

---

## 4. أدوات البناء والتطوير

### 4.1 Java Ecosystem

| الأداة | الوصف | Offline |
|-------|-------|---------|
| **Maven** 📦 | بناء Java | 🔄 repo محلي |
| **Gradle** 📦 | بناء متقدم | 🔄 cache |
| **JFrog Artifactory** 📦 | مستودع artifacts | ✅ |
| **Nexus Repository** 📦 | مستودع artifacts | ✅ |

#### Maven Offline Repository

```bash
# على جهاز متصل - إنشاء repository محلي
mvn dependency:go-offline

# نسخ local repository
tar -czf maven-repo.tar.gz ~/.m2/repository

# على جهاز Offline
tar -xzf maven-repo.tar.gz -C ~/
mv repository ~/.m2/

# إعداد Maven للعمل offline
mvn -o package  # -o = offline mode
```

#### Nexus Repository Manager

```bash
# على جهاز متصل
docker pull sonatype/nexus3:latest
docker save sonatype/nexus3 -o nexus3.tar

# على جهاز Offline
docker load -i nexus3.tar

docker run -d \
  --name nexus \
  -p 8081:8081 \
  -p 8082:8082 \
  -v nexus-data:/nexus-data \
  sonatype/nexus3

# الوصول: http://localhost:8081
# Default: admin / استخراج من: docker exec nexus cat /nexus-data/admin.password

# إعداد proxy للمستودعات الخارجية
# Maven Central, npm, PyPI, Docker Hub, etc.
```

### 4.2 Node.js Ecosystem

| الأداة | الوصف | Offline |
|-------|-------|---------|
| **npm** 📦 | مدير حزم Node | 🔄 |
| **yarn** 📦 | مدير حزم بديل | 🔄 |
| **pnpm** 📦 | مدير حزم فعال | 🔄 |
| **Verdaccio** 📦 | npm registry خاص | ✅ |

#### Verdaccio - npm Registry محلي

```bash
# على جهاز متصل
npm install -g verdaccio
verdaccio

# على جهاز Offline
# تثبيت Verdaccio
npm install -g verdaccio

# تشغيل
verdaccio

# إعداد npm للاستخدام registry محلي
npm set registry http://localhost:4873/

# نشر حزم محلياً
npm publish --registry http://localhost:4873/
```

### 4.3 Python Ecosystem

| الأداة | الوصف | Offline |
|-------|-------|---------|
| **pip** 📦 | مدير حزم Python | 🔄 |
| **poetry** 📦 | إدارة dependencies | 🔄 |
| **pipenv** 📦 | virtualenv + pip | 🔄 |
| **devpi** 📦 | PyPI server خاص | ✅ |

#### PyPI Mirror محلي

```bash
# على جهاز متصل - تحميل الحزم
pip download -r requirements.txt -d ~/pypi-packages/

# على جهاز Offline
pip install --no-index --find-links ~/pypi-packages/ -r requirements.txt

# أو استخدام devpi
pip install devpi-server
devpi-server --init
devpi-server --start

# تحميل packages لـ devpi
devpi use http://localhost:3141
devpi login root --password=''
devpi index -c dev
devpi use root/dev
devpi upload
```

---

## 5. أدوات الاختبار

### 5.1 أدوات الاختبار

| الأداة | اللغة | Offline |
|-------|-------|---------|
| **JUnit** 📦 | Java | ✅ |
| **pytest** 📦 | Python | ✅ |
| **Jest** 📦 | JavaScript | ✅ |
| **Selenium** 📦 | Multi | 🔄 drivers |
| **Cypress** 📦 | JavaScript | 🔄 |
| **Postman** 📦 | API Testing | ✅ |
| **k6** 📦 | Load Testing | ✅ |
| **JMeter** 📦 | Load Testing | ✅ |
| **Gatling** 📦 | Load Testing | ✅ |

---

## 6. Service Mesh و API Gateway

### 6.1 Service Mesh

| الأداة | الوصف | Offline |
|-------|-------|---------|
| **Istio** 📦 | service mesh كامل | ✅ |
| **Linkerd** 📦 | service mesh خفيف | ✅ |
| **Consul** 📦 | service mesh + discovery | ✅ |

### 6.2 API Gateway

| الأداة | الوصف | Offline |
|-------|-------|---------|
| **Kong** 📦 | API gateway | ✅ |
| **Traefik** 📦 | reverse proxy حديث | ✅ |
| **NGINX** 📦 | reverse proxy كلاسيكي | ✅ |
| **HAProxy** 📦 | load balancer | ✅ |
| **Envoy** 📦 | proxy متقدم | ✅ |

---

## 7. إدارة المشاريع والتعاون

### 7.1 أدوات إدارة المشاريع

| الأداة | الوصف | Offline |
|-------|-------|---------|
| **Jira** ⚠️ | إدارة مشاريع | Jira Server/DC |
| **GitLab Issues** 📦 | مدمج في GitLab | ✅ |
| **Redmine** 📦 | إدارة مشاريع | ✅ |
| **Taiga** 📦 | Agile management | ✅ |
| **Plane** 📦 | بديل Jira مفتوح | ✅ |

### 7.2 التعاون والتوثيق

| الأداة | الوصف | Offline |
|-------|-------|---------|
| **Confluence** ⚠️ | wiki | Confluence Server |
| **Wiki.js** 📦 | wiki حديث | ✅ |
| **BookStack** 📦 | documentation | ✅ |
| **Mattermost** 📦 | بديل Slack | ✅ |
| **Rocket.Chat** 📦 | chat platform | ✅ |

---

## 8. منصات السحابة

### 8.1 الخدمات السحابية

| المنصة | الاتصال | حل Offline |
|--------|---------|-----------|
| **AWS** ⚠️ | إنترنت | LocalStack, MinIO |
| **Azure** ⚠️ | إنترنت | Azurite (storage emulator) |
| **GCP** ⚠️ | إنترنت | محدود |

#### LocalStack - AWS محلي

```bash
# على جهاز متصل
docker pull localstack/localstack:latest
docker save localstack/localstack -o localstack.tar

# على جهاز Offline
docker load -i localstack.tar

# تشغيل LocalStack
docker run -d \
  --name localstack \
  -p 4566:4566 \
  -p 4510-4559:4510-4559 \
  -e SERVICES=s3,dynamodb,lambda,sqs,sns \
  -v /var/run/docker.sock:/var/run/docker.sock \
  localstack/localstack

# استخدام AWS CLI مع LocalStack
aws --endpoint-url=http://localhost:4566 s3 mb s3://my-bucket
aws --endpoint-url=http://localhost:4566 s3 ls
```

#### MinIO - S3 متوافق

```bash
# على جهاز متصل
wget https://dl.min.io/server/minio/release/linux-amd64/minio
wget https://dl.min.io/client/mc/release/linux-amd64/mc

# على جهاز Offline
sudo mv minio /usr/local/bin/
sudo mv mc /usr/local/bin/
sudo chmod +x /usr/local/bin/minio /usr/local/bin/mc

# تشغيل MinIO
export MINIO_ROOT_USER=admin
export MINIO_ROOT_PASSWORD=password123
minio server /data --console-address ":9001"

# إعداد client
mc alias set myminio http://localhost:9000 admin password123
mc mb myminio/mybucket
mc cp myfile.txt myminio/mybucket/
```

---

## 9. استراتيجية Offline Toolkit الشاملة

### 9.1 البنية التحتية الأساسية

```
┌─────────────────────────────────────────────────────────┐
│           Offline DevOps Infrastructure                  │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  [Git Server]          [Docker Registry]                │
│  GitLab CE/Gitea       Local Registry                   │
│                                                          │
│  [CI/CD]               [Artifact Repository]            │
│  Jenkins/GitLab CI     Nexus/Artifactory                │
│                                                          │
│  [Container Platform]  [Monitoring]                     │
│  Kubernetes (k3s)      Prometheus + Grafana             │
│                                                          │
│  [Secrets Management]  [Logging]                        │
│  HashiCorp Vault       ELK Stack                        │
│                                                          │
│  [Configuration]       [Databases]                      │
│  Ansible               PostgreSQL, Redis                │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### 9.2 قائمة التحقق Offline Readiness

#### مرحلة 1: التحضير (على جهاز متصل)

```bash
#!/bin/bash
# offline-devops-prep.sh

mkdir -p ~/offline-toolkit/{binaries,images,repos,scripts}

# 1. تحميل binaries الأساسية
cd ~/offline-toolkit/binaries
wget <URLs لجميع الأدوات>

# 2. تحميل Docker images
CRITICAL_IMAGES=(
    "gitlab/gitlab-ce:latest"
    "postgres:16"
    "redis:7"
    "nginx:latest"
    "jenkins/jenkins:lts"
    "registry:2"
    "sonatype/nexus3:latest"
    "prom/prometheus:latest"
    "grafana/grafana:latest"
    "docker.elastic.co/elasticsearch/elasticsearch:8.12.2"
    "docker.elastic.co/kibana/kibana:8.12.2"
)

cd ~/offline-toolkit/images
for image in "${CRITICAL_IMAGES[@]}"; do
    filename=$(echo $image | tr '/:' '_')
    docker pull $image
    docker save $image -o ${filename}.tar
done

# 3. مزامنة package repositories
mkdir -p ~/offline-toolkit/repos/{ubuntu,rocky,pypi,npm}

# Ubuntu/Debian
cd ~/offline-toolkit/repos/ubuntu
apt-mirror  # بعد إعداد mirror.list

# Rocky/RHEL
cd ~/offline-toolkit/repos/rocky
reposync --download-metadata --repoid=baseos
reposync --download-metadata --repoid=appstream

# PyPI
pip download -r common-requirements.txt -d ~/offline-toolkit/repos/pypi/

# npm
cd ~/offline-toolkit/repos/npm
npm install -g verdaccio
# مزامنة الحزم الشائعة

# 4. تحميل Kubernetes components
cd ~/offline-toolkit/binaries
# kubectl, kubeadm, kubelet
# images: kube-apiserver, kube-controller-manager, etc.

# 5. ضغط كل شيء
cd ~/offline-toolkit
tar -czf devops-toolkit-$(date +%Y%m%d).tar.gz .

echo "✅ Offline toolkit ready!"
echo "Size: $(du -sh devops-toolkit-*.tar.gz)"
```

#### مرحلة 2: النقل

```bash
# نقل الملف الكبير
# خيارات:
# 1. Hard drive خارجي
# 2. USB كبير (256GB+)
# 3. شبكة داخلية
# 4. DVD/Blu-ray متعدد

# التحقق من السلامة
sha256sum devops-toolkit-*.tar.gz > checksum.txt
```

#### مرحلة 3: التثبيت (على الشبكة Offline)

```bash
#!/bin/bash
# offline-devops-install.sh

# 1. فك الضغط
tar -xzf devops-toolkit-*.tar.gz -C /opt/devops-toolkit

# 2. تثبيت binaries
cd /opt/devops-toolkit/binaries
for binary in *; do
    sudo cp $binary /usr/local/bin/
    sudo chmod +x /usr/local/bin/$binary
done

# 3. تحميل Docker images
cd /opt/devops-toolkit/images
for tar_file in *.tar; do
    docker load -i $tar_file
done

# 4. إعداد repositories المحلية
# Ubuntu
sudo mkdir -p /srv/repos/ubuntu
sudo cp -r /opt/devops-toolkit/repos/ubuntu/* /srv/repos/ubuntu/

# Rocky
sudo mkdir -p /srv/repos/rocky
sudo cp -r /opt/devops-toolkit/repos/rocky/* /srv/repos/rocky/

# 5. إعداد الخدمات الأساسية
./setup-gitlab.sh
./setup-jenkins.sh
./setup-docker-registry.sh
./setup-nexus.sh
./setup-monitoring.sh

echo "✅ Offline DevOps infrastructure installed!"
```

### 9.3 Scripts الإعداد الآلي

#### setup-core-services.sh

```bash
#!/bin/bash
# Setup Core DevOps Services Offline

set -e

INSTALL_DIR="/opt/devops"
DATA_DIR="/data/devops"

echo "🚀 Setting up Offline DevOps Core Services..."

# 1. GitLab
echo "📦 Setting up GitLab..."
docker run -d \
  --name gitlab \
  --hostname gitlab.local \
  -p 443:443 -p 80:80 -p 22:22 \
  -v ${DATA_DIR}/gitlab/config:/etc/gitlab \
  -v ${DATA_DIR}/gitlab/logs:/var/log/gitlab \
  -v ${DATA_DIR}/gitlab/data:/var/opt/gitlab \
  --restart always \
  gitlab/gitlab-ce:latest

# 2. Docker Registry
echo "📦 Setting up Docker Registry..."
docker run -d \
  --name registry \
  -p 5000:5000 \
  -v ${DATA_DIR}/registry:/var/lib/registry \
  --restart always \
  registry:2

# 3. Nexus Repository
echo "📦 Setting up Nexus..."
docker run -d \
  --name nexus \
  -p 8081:8081 \
  -v ${DATA_DIR}/nexus:/nexus-data \
  --restart always \
  sonatype/nexus3

# 4. Jenkins
echo "📦 Setting up Jenkins..."
docker run -d \
  --name jenkins \
  -p 8080:8080 -p 50000:50000 \
  -v ${DATA_DIR}/jenkins:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  --restart always \
  jenkins/jenkins:lts

# 5. Prometheus + Grafana
echo "📦 Setting up Monitoring..."
docker-compose -f monitoring-stack.yml up -d

# 6. Vault
echo "📦 Setting up HashiCorp Vault..."
docker run -d \
  --name vault \
  -p 8200:8200 \
  -e VAULT_DEV_ROOT_TOKEN_ID=root \
  -v ${DATA_DIR}/vault:/vault/data \
  --cap-add=IPC_LOCK \
  --restart always \
  vault:latest

# 7. PostgreSQL (shared DB)
echo "📦 Setting up PostgreSQL..."
docker run -d \
  --name postgres \
  -e POSTGRES_PASSWORD=devops123 \
  -v ${DATA_DIR}/postgres:/var/lib/postgresql/data \
  -p 5432:5432 \
  --restart always \
  postgres:16

# 8. Redis (shared cache)
echo "📦 Setting up Redis..."
docker run -d \
  --name redis \
  -v ${DATA_DIR}/redis:/data \
  -p 6379:6379 \
  --restart always \
  redis:7

echo "✅ Core services deployed!"
echo ""
echo "Access URLs:"
echo "  GitLab:   http://localhost"
echo "  Registry: http://localhost:5000"
echo "  Nexus:    http://localhost:8081"
echo "  Jenkins:  http://localhost:8080"
echo "  Vault:    http://localhost:8200"
```

---

## 10. خارطة طريق التنفيذ

### المرحلة 1: الأساسيات (الأسبوع 1-2)

```
✅ تثبيت نظام التشغيل (Ubuntu 24.04 أو Rocky 10)
✅ تثبيت Docker + Docker Compose
✅ إعداد GitLab CE
✅ إعداد Docker Registry
✅ إعداد Nexus Repository
```

### المرحلة 2: CI/CD (الأسبوع 3-4)

```
✅ إعداد Jenkins
✅ إنشاء pipelines أساسية
✅ ربط GitLab + Jenkins
✅ إعداد GitLab CI/CD Runners
```

### المرحلة 3: Orchestration (الأسبوع 5-6)

```
✅ تثبيت Kubernetes (k3s)
✅ إعداد Helm
✅ نشر تطبيقات على K8s
```

### المرحلة 4: المراقبة (الأسبوع 7-8)

```
✅ إعداد Prometheus
✅ إعداد Grafana
✅ إعداد ELK Stack
✅ إنشاء dashboards
```

### المرحلة 5: الأمان والأتمتة (الأسبوع 9-10)

```
✅ إعداد Vault
✅ إعداد Ansible
✅ إعداد Terraform
✅ تطبيق security scanning
```

---

## 📊 المتطلبات النهائية

### الموارد المطلوبة

| المكون | CPU | RAM | Storage |
|--------|-----|-----|---------|
| **GitLab CE** | 4 cores | 8 GB | 50 GB |
| **Jenkins** | 2 cores | 4 GB | 20 GB |
| **Kubernetes** | 4+ cores | 8+ GB | 100+ GB |
| **Monitoring** | 2 cores | 4 GB | 50 GB |
| **Nexus** | 2 cores | 4 GB | 100+ GB |
| **Registry** | 1 core | 2 GB | 100+ GB |
| **Vault** | 1 core | 2 GB | 10 GB |
| **Databases** | 2 cores | 4 GB | 50 GB |
| **Total (Minimum)** | 16 cores | 32 GB | 480 GB |
| **Recommended** | 32 cores | 64 GB | 1 TB |

### حجم التنزيل الإجمالي

```
- Binaries: ~10 GB
- Docker Images: ~50 GB
- Package Repositories: ~150-200 GB
- Documentation & Scripts: ~1 GB
─────────────────────────────────
Total: ~210-260 GB
```

---

**✅ تم اكتمال الدليل الشامل لأدوات DevOps مع حلول Offline**

**الملفات:**
1. sysadmin-tools-ubuntu24.md ✓
2. sysadmin-tools-rocky10.md ✓
3. devops-tools-part1.md ✓
4. devops-tools-part2.md ✓

جميع الملفات جاهزة للتحميل المباشر! 🎉

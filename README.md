# Complete Monitoring Setup: Prometheus + Node Exporter + Flask App + Grafana on AWS EC2

This project demonstrates how to install and configure a full monitoring stack on an AWS EC2 instance using:

- **Prometheus** – Metrics collection and monitoring
- **Node Exporter** – System-level metrics
- **Flask Application** – Custom application metrics
- **Grafana** – Visualization dashboards

---

## ➤ Architecture Overview

EC2 (Ubuntu)
│
├── Node Exporter (Port 9100)
├── Python Flask App (Port 5000)
│
└── Prometheus (Port 9090)
        │
        └── Grafana (Port 3000)

---

# ➤ Step 1: Launch EC2 Instance

- Instance Type: `t3.micro`
- Operating System: Ubuntu 22.04 LTS
- Create or select a key pair

---

# ➤ Step 2: Configure Security Group (Inbound Rules)

Allow the following ports:

| Port | Purpose |
|------|----------|
| 22   | SSH |
| 9090 | Prometheus |
| 9100 | Node Exporter |
| 5000 | Python Application |
| 3000 | Grafana |

Source: `0.0.0.0/0`

---

# ➤ Step 3: Connect to EC2

```bash
ssh -i your-key.pem ubuntu@your-public-ip
```
Update system:
```
sudo apt update -y
```
# ➤ Step 4: Install Prometheus

## 1️⃣ Create User and Directories

```bash
sudo useradd --no-create-home --shell /bin/false prometheus
sudo mkdir /etc/prometheus
sudo mkdir /var/lib/prometheus
sudo chown prometheus:prometheus /var/lib/prometheus
```

---

## 2️⃣ Download Prometheus

```bash
cd /tmp
wget https://github.com/prometheus/prometheus/releases/download/v2.53.1/prometheus-2.53.1.linux-amd64.tar.gz
tar -xvf prometheus-2.53.1.linux-amd64.tar.gz
cd prometheus-2.53.1.linux-amd64
```

---

## 3️⃣ Move Required Files

```bash
sudo mv prometheus /usr/local/bin/
sudo mv promtool /usr/local/bin/
sudo mv consoles /etc/prometheus
sudo mv console_libraries /etc/prometheus
sudo mv prometheus.yml /etc/prometheus
```

---

## 4️⃣ Set Ownership

```bash
sudo chown -R prometheus:prometheus /etc/prometheus
sudo chown prometheus:prometheus /usr/local/bin/prometheus
```

---

## 5️⃣ Create Prometheus Systemd Service

```bash
sudo vim /etc/systemd/system/prometheus.service
```

Paste the following:

```ini
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
--config.file=/etc/prometheus/prometheus.yml \
--storage.tsdb.path=/var/lib/prometheus \
--web.console.templates=/etc/prometheus/consoles \
--web.console.libraries=/etc/prometheus/console_libraries

[Install]
WantedBy=multi-user.target
```

---

## 6️⃣ Start and Enable Prometheus

```bash
sudo systemctl daemon-reload
sudo systemctl start prometheus
sudo systemctl enable prometheus
sudo systemctl status prometheus
```

Access Prometheus:

```
http://YOUR_PUBLIC_IP:9090
```

---

# ➤ Step 5: Install Node Exporter

## Download and Install

```bash
cd /tmp
wget https://github.com/prometheus/node_exporter/releases/download/v1.8.1/node_exporter-1.8.1.linux-amd64.tar.gz
tar -xvf node_exporter-1.8.1.linux-amd64.tar.gz
sudo mv node_exporter-1.8.1.linux-amd64/node_exporter /usr/local/bin/
sudo useradd -rs /bin/false node_exporter
```

---

## Create Node Exporter Service

```bash
sudo vim /etc/systemd/system/node_exporter.service
```

Paste:

```ini
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
```

---

## Start and Enable Node Exporter

```bash
sudo systemctl daemon-reload
sudo systemctl start node_exporter
sudo systemctl enable node_exporter
sudo systemctl status node_exporter
```

Verify:

```
http://YOUR_PUBLIC_IP:9100/metrics
```

---

# ➤ Step 6: Configure Prometheus to Scrape Node Exporter

Edit configuration:

```bash
sudo vim /etc/prometheus/prometheus.yml
```

Under `scrape_configs`, add:

```yaml
- job_name: 'node_exporter'
  static_configs:
    - targets: ['localhost:9100']
```

Restart Prometheus:

```bash
sudo systemctl restart prometheus
```

---

# ➤ Step 7: Deploy Python Application with Custom Metrics

## Install Python Dependencies

```bash
sudo apt install python3-pip python3-venv -y
python3 -m venv myenv
source myenv/bin/activate
pip install flask prometheus_client
```

---

## Create `app.py`

```bash
vim app.py
```

Paste:

```python
from flask import Flask
from prometheus_client import Counter, generate_latest, CONTENT_TYPE_LATEST

app = Flask(__name__)

REQUESTS = Counter('hello_world_total', 'Total Hello World Requests')

@app.route('/')
def hello():
    REQUESTS.inc()
    return "Hello, World!"

@app.route('/metrics')
def metrics():
    return generate_latest(), 200, {'Content-Type': CONTENT_TYPE_LATEST}

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

---

## Run the Application

```bash
nohup python3 app.py > app.log 2>&1 &
```

Test:

```
http://YOUR_PUBLIC_IP:5000
```

Refresh multiple times to increase the counter.

---

# ➤ Step 8: Add Python App to Prometheus

Edit configuration:

```bash
sudo vim /etc/prometheus/prometheus.yml
```

Add:

```yaml
- job_name: 'python_app'
  static_configs:
    - targets: ['localhost:5000']
```

Restart Prometheus:

```bash
sudo systemctl restart prometheus
```

---

# ➤ Step 9: Verify Metrics in Prometheus

Open Prometheus:

```
http://YOUR_PUBLIC_IP:9090
```

Search:

```
hello_world_total
```

You should see the counter value increasing.

---

# ➤ Step 10: Show CPU Usage Metric

Run the following query in Prometheus UI:

```promql
100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

This displays CPU usage percentage.

---

# ➤ Install Grafana

```bash
sudo apt-get install -y apt-transport-https software-properties-common wget
sudo mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
sudo apt-get update
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee /etc/apt/sources.list.d/grafana.list
sudo apt-get install grafana -y
sudo systemctl start grafana-server
sudo systemctl enable grafana-server
```

Access Grafana:

```
http://YOUR_PUBLIC_IP:3000
```

Default Login:

- Username: `admin`
- Password: `admin`

---

# ➤ Add Prometheus Data Source in Grafana

1. Go to **Connections → Data Sources**
2. Select **Prometheus**
3. Set URL to:

```
http://localhost:9090
```

4. Click **Save & Test**

---

# ➤ Final Outcome

✔ Prometheus collecting system metrics  
✔ Node Exporter monitoring EC2 resources  
✔ Flask app exposing custom metrics  
✔ Grafana visualizing metrics through dashboards  

Your monitoring stack is now fully deployed and operational
